---
title: "Setting up Kubernetes Cluster Infrastructure" 
categories:
  - Blog
tags:
  - cluster
  - pi
  - kubernetes
  - k3s
  - flux
---

> Setting up a load balancer, ingress controller, wildcard tls, distributed storage, and rancher!

When running Kubernetes on bare metal, key infrastructure is missing out of the box. 

* Load Balancer
* Ingress Controller
* TLS Certificate Management
* Distributed Storage

All manifests are available in my [pikluster repository](https://github.com/dvignoles/pikluster/tree/main/cluster)! 

## Metallb

[Metallb](https://metallb.universe.tf/) is a network load balancer implementation. Normally your load balancer is provided by the cloud provider your cluster is running on. You can also of course run your own load balancer outside of the cluster, which is probably a better solution in terms of high availability. To use metallb, you allocate a pool of ip addresses for the loadbalancer to distribute as needed. When addresses from the pool are allocated, they are externally announced via ARP or BGP. 

As I'm using my home network, I opted to just shorten my router's DHCP range and allocate the excess addresses to metallb. 

DHCP range before: `192.168.50.2-192.168.50.254`

DHCP range after: `192.168.50.2-192.168.50.246`

That leaves metallb with the `192.168.50.247-192.168.50.255` range to play with.

Let's follow the [installation instructions](https://metallb.universe.tf/installation/#installation-with-helm)!

The documentation recommends some pod security admissions for the namespace. 

```sh
kubectl apply -f core/namespaces/networking.yaml
```

Let's install the helm chart. 

```sh
helm repo add metallb https://metallb.github.io/metallb
helm install -n networking --create-namespace networking metallb/metallb
```

Metallb is installed, but *not configured*.

```sh
kubectl get pods -n networking
```
```
NAME                                  READY   STATUS    RESTARTS        AGE
metallb-speaker-zc5cc                 1/1     Running   2 (6h20m ago)   27h
metallb-controller-585c5cd8c7-psrv6   1/1     Running   2 (6h18m ago)   27h
metallb-speaker-hs65x                 1/1     Running   2 (6h18m ago)   27h
metallb-speaker-wljgg                 1/1     Running   2 (6h18m ago)   27h
metallb-speaker-l5x57                 1/1     Running   2 (6h18m ago)   27h
```

We need to allocate & advertise the pool.

```sh
kubectl apply -f core/networking/metallb/ipaddresspool.yaml
```

We can now expose a service of type load balancer!

```sh
# example
kubectl expose deploy nginx --port 80 --type LoadBalancer
kubectl get svc
```

You should see an `EXTERNAL-IP` listed for the service in our ip address range!

## Cert-Manager

In order to manage TLS certificates, we'll use a certificate controller to provision and update certificates for the cluster. Specifically, we'll provision a wild card cerficate from lets-encrypt for TLS on our local network. To obtain a certificate, we need to first own a domain which cert-manager can then verify. 

```yaml
# values.yaml
installCRDs: true
replicaCount: 3
extraArgs:
    - --dns01-recursive-nameservers=1.1.1.1:53,9.9.9.9:53
    - --dns01-recursive-nameservers-only
podDnsPolicy: None
podDnsConfig:
    nameservers:
    - "1.1.1.1"
    - "9.9.9.9"
```

```sh
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --values=values.yaml
```

### CloudFlare DNS

My domain, danielvignoles.com, is currently managed via cloudflare. Cert-manager has guides for various issuers, but to use cloudflare we'll follow [this documentation](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/). 

Issue an api token from the cloudflare dashboard with `Zone.DNS` permissions. Add your token as a kubernetes secret!

```sh
 kubectl -n cert-manager create secret generic cloudflare-token-secret \
--from-literal=cloudflare-token=<TOKEN> \
--dry-run=client \
--type=opague \
-o yaml > cloudflare-token-secret.yaml

kubectl apply -f cloudflare-token-secret.yaml
```

Check the secret exists with `kubectl get secrets -n cert-manager`. 

We can now setup a certificate issuer which references the `cloudflare-token` secret.

```sh
kubectl apply -f core/cert-manager/issuers/letsencrypt-production.yaml
```

Before issuing any certs, let's set up our ingress controller. 

## Traefik

An ingress controller acts as a reverse proxy and load balancer for layer 7 traffic. It defines how external traffic is routed within the cluster based on configuration options. In this configuration we expose the ingress contoller as a type load balancer service and route all traffic through this ip. This lets us use one single ip address to reach all services within the cluster. The controller we are using, traefik, also manages security middleware and tls configuration. 

```yaml
# values.yaml
globalArguments:
  - "--global.sendanonymoususage=false"
  - "--global.checknewversion=false"

additionalArguments:
  - "--serversTransport.insecureSkipVerify=true"
  - "--log.level=INFO"

deployment:
  enabled: true
  replicas: 3
  annotations: {}
  podAnnotations: {}
  additionalContainers: []
  initContainers: []

ports:
  web:
    redirectTo: websecure
  websecure:
    tls:
      enabled: true

ingressRoute:
  dashboard:
    enabled: false

providers:
  kubernetesCRD:
    enabled: true
    ingressClass: traefik-external
    allowCrossNamespace: true # allows crossnamespace middleware
  kubernetesIngress:
    enabled: true
    publishedService:
      enabled: false

rbac:
  enabled: true

service:
  enabled: true
  type: LoadBalancer
  annotations: {}
  labels: {}
  spec:
    loadBalancerIP: 192.168.50.254 # IP in metallb ipaddresspool
  loadBalancerSourceRanges: []
  externalIPs: []
```

```sh
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install --namespace=traefik traefik traefik/traefik --values=values.yaml
```

The `providers.kubernetesCRD.allowCrossNamespace: true` makes it possible to set up some default middleware we can use for traffic in all namespaces. Let's set up some default good practice headers for all traffic as well a basic authentication middleware for local network traffic. 

```sh
# requires secret for basic auth credentials
kubectl apply -f core/networking/traefik/default-basic-auth.yaml
kubectl apply -f core/traefik/default-headers.yaml
```

###  Wildcard certificate

In our setup, we will provision a default wildcard certificate for our traefik ingress. All requests matching the `*.local.danielvignoles.com` wildcard will work over https if coming through our ingress. Due to how secrets work in kubernetes, the certifcate must be provisioned for the namespace where traefik is running.

Traefik can then use this certificate as it's [default certificate](https://doc.traefik.io/traefik/https/tls/#default-certificate).

`kubectl apply -f core/networking/certificates/local-danielvignoles-com.yaml`

Watch the challenges status. It may take a while, but eventually the dns01 challenge should succeed. 

```sh
kubectl get challenges --all-namespaces
```

When your certificate is ready you can examine the custom resource and underlying secret.

```sh
kubectl get certifcate -n networking
```
```
NAME                       READY   SECRET                         AGE
local-danielvignoles-com   True    local-danielvignoles-com-tls   25h
```

We have our certificate! Let's set this as the default certificate for traefik.

```sh
kubectl apply -f core/networking/traefik/default-certificate.yaml
```

And setup our first ingress route, to view the traefik dashboard! 

```sh
kubectl apply -f core/networking/traefik/dashboard/ingress.yaml
```

As `traefik.local.danielvignoles.com` matches our wildcard certificate we have tls!

### Local DNS
This is technically outside the scope of the kubernetes cluster, but it's worth addressing *how* to actually make use of ingress entrypoints.

On my network I have an old raspberry pi 3b+ directly linked to my router acting as a local DNS server using [pi-hole](https://pi-hole.net/). This local DNS is backed up by a public DNS provider (like 8.8.8.8) in case no local defintion is found. The local DNS functionality of pi-hole is super convenient for our ingress use case. If we set the the pi-hole as our nameserver (either on our clients or on our home router), we can associate our local domain names with the ingress controller IP address. Our local DNS forwards traffic to our ingress controller, which can then connect with the correct service based on the hostname used. 

If you're working on linux and just want to test out the ingress route, you can also override DNS by adding an entry in `/etc/hosts`. 

```
# /etc/hosts
192.168.50.254 traefik.local.danielvignoles.com
```

## Longhorn
The default storage class for k3s is a local path storage provisioner. This solution doesn't guarantee that a pod re-scheduled to a different node will have access to the same volumes. It also doesn't provide any redundancy or backups. We are using [Longhorn](https://longhorn.io/) as an alternative, a distributed block storage solution. Using longhorn, all data is synchronously replicated between multiple nodes. Longhorn also provides a system for snapshots and backups to external storage such as NFS or S3.

Longhorn requires each node the cluster meets its [installation requirements](https://longhorn.io/docs/1.3.0/deploy/install/#installation-requirements). You can check if you've met the requirements with their environment checking script.

```sh
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.3.0/scripts/environment_check.sh | bash
```

Once all the dependencies are installed its pretty straightforward to install with helm. 

```sh
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
```

Longhorn has a nice dashboard for managing the distributed storage and configuring backups/snapshots. We'll also set up an ingress route to access this service.

```sh
kubectl apply -f core/longhorn-system/ingress.yaml
```

```sh
kubectl get storageclass
```
```
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
longhorn (default)     driver.longhorn.io      Delete          Immediate              true                   27h
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  33h
```

It doesn't make sense to have two default storage classes, so let's patch the secondary local-path option.

```sh
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

## Bonus: Rancher

Infrastructure as code is great, but I also want a gui to fall back on for testing and exploration. [Rancher](https://rancher.com/) gives us that gui to examine, create, and transform kubernetes resources. 

```sh
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.local.danielvignoles.com \
  --set bootstrapPassword=admin \
  --set tls=external

# monitor rollout progress
kubectl -n cattle-system rollout status deploy/rancher
```

We'll let our traefik ingress route handle TLS.

```sh
kubectl apply -f core/cattle-sytem/ingress.yaml
```

We have rancher!

Next: [Part 3: GitOps with Flux](/blog/post-flux-gitops)