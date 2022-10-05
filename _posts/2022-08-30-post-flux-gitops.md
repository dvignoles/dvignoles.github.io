---
title: "GitOps Infrastructure with Flux!" 
categories:
  - Blog
tags:
  - cluster
  - pi
  - kubernetes
  - flux
  - gitops
---

> How to use GitOps for kubernetes using Flux!

GitOps brings DevOps best practices, most notably version control, to your kubernetes infrastructure. The basic idea is that you define the desired state of your infrastructure in a git repository, and a controller reconciles your cluster with that repository accordingly. After wiping and recreating my cluster infrastructure a few times, I knew I wanted to learn this technology.

There are a number of GitOps tools available. During my research I looked at Flux, Fleet by Rancher, and ArgoCD. I'm sure any of these would work great for my simple use case. I decided to use Flux just because it seemed to have a "no-nonsense" CLI approach to things. In retrospect I might have been better off with ArgoCD, which seems like a somewhat more mature user-friendly project. Regardless, Flux is super cool and works great! It took a lot of trial and error, but my flux managed infrastructure is up and running smoothly now. There's a lot to it, and I've tried to touch on some of primary pain points below. 

## Flux

To get started, [Install the Flux CLI](https://fluxcd.io/docs/installation/).

```sh
# yolo pipe
curl -s https://fluxcd.io/install.sh | sudo bash
```

Bootstrap flux managed cluster.

```sh
 flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=dvignoles \
  --repository=flux \
  --branch=main \
  --path=cluster/base \
  --personal \
  --token-auth
```

The above command requires a [github token](https://github.com/settings/tokens) to manage repositories. [My flux repository](https://github.com/dvignoles/flux) is now ready to go! A single flux deployment/repo can manage multiple clusters, but we will be managing just one cluster rooted in `cluster/base`. In a professional setting you might set up a folder structure for differentiating between staging and production. 

Do a `kubectl -n flux-system get pods` and you should see the various flux controllers running. To engage with our cluster via flux, simply push manifests to your flux repo! Flux reconciles the updated source with the current state of the cluster. When you push a change you can monitor this reconciliation.

```sh
flux get kustomizations --watch
```

## Secrets

Even in a private git repository, you probably shouldn't store any secret passwords or API keys in plain text. You can of course just manage secrets outside of GitOps manually. If you do want to manage secrets declaratively, flux provides integrations for storing encrypted secrets in your repository. I tried out both [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) and [Mozilla SOPS](https://github.com/mozilla/sops). Both workflows work fine, but I found SOPS a bit more intuitive so that's what I'm going with. You'll need to install SOPS and whatever encryption backend you are using. I'm using [age](https://github.com/FiloSottile/age), but use whatever suits you. Get both the `age` and `sops` binaries installed by following along their readmes. 

First, generate your age key pair. Remember to also backup your private key somewhere safe!
```sh
age-keygen -o age.agekey

# add private key to kubernetes for decryption.
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

Create a .sops.yaml file at the root of your repository. Paste your public key in. `sops` will look for this file when doing encryption. It's safe to push your public key to your repository. 

```yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1etfpwhwrvmd7xqh5a8yn6xx8l0q8e0rh2ca0aaf70yeeawlll9qseh2ljw
```

Let's encrypt this demo secret. 

```yaml
# basic-auth.yaml
apiVersion: v1
data:
  #  htpasswd -nb <user> <pass> | openssl base64
  users: YWRtaW46JGFwcjEkMlBIZTlFazEkVGtqeVZoNlBhdjlpeGxWSUtDRTFXMAoK
kind: Secret
metadata:
  name: basic-auth
  namespace: default
```

```sh
sops --encrypt basic-auth.yaml > basic-auth.sops.yaml
```

```yaml
# basic-auth.sops.yaml
apiVersion: v1
data:
    #ENC[AES256_GCM,data:ENqPfz2bMSxcPp+QT/Q/TWXHTVLp4t4s2gDbvVxwJg+gVvoNUZdkbmg=,iv:8YTW1+Ip0fG+M/apRQlxQFGHT9NzefmQcaZZHMyp5xE=,tag:iAD/SJbEhSsVpaabihouvg==,type:comment]
    users: ENC[AES256_GCM,data:Fzg9VpCkTV0pFX838FGjtUU84z8umJNcmpH0XDA6+pJgAyxz3GYcfhRcKwnHFiksUtUJyiw3JtVizIuH,iv:kUQx43neztD72UdlP4Lh9j00aiGF1s4MUwwOFTogj48=,tag:iVDMf6uLwe5CFzYTn7nejg==,type:str]
kind: Secret
metadata:
    name: basic-auth
    namespace: default
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age:
        - recipient: age1etfpwhwrvmd7xqh5a8yn6xx8l0q8e0rh2ca0aaf70yeeawlll9qseh2ljw
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSB5SndBdS9HQ1E5QklrSTlF
            dUVqV0xLMmJNT25FbURkZVhTcUtlejNSeVdNCk9kWDVtSGlteU9jTjV0Q0hwSzNU
            bnFRWGk0SUxYQkYybkZ5YnVnNjZoSlUKLS0tIGZRMWlHV1ZiMlpkdGtsVkZRNkZG
            Y1pOWDZwM3RNWk1mNnhtVVA5cDhuKzQKOaJI2gabvxW8JfulOnQQ3RUHBfhAzG1R
            AlQwS9jWmJ0T2PsoVl4y9PgQbYCDj4GTAsp6sMdkzFvgQq4NNmweww==
            -----END AGE ENCRYPTED FILE-----
    lastmodified: "2022-09-22T00:47:09Z"
    mac: ENC[AES256_GCM,data:otL1WG+Heet3pD2wcfhcRJiZA9lfjQa+GELqfobSBtZa+ezjzmijk8WXQAMtE2KTeMujpbsuEQn83A5Zqz/9sd5VT/vK1i71/Ge2SNgBX/nCBw3YM8HvqE9XCWscGo85fg8BKCCYpbGPD0jzvo7xyUKzZojKyxwQDcQKtIKZfQ4=,iv:LWLFYYSb+znveWUtODoIWzXkztsp4/fqjzY8vgghhSU=,tag:V5pIFLFhLVX1oXNkMagD3w==,type:str]
    pgp: []
    encrypted_regex: ^(data|stringData)$
    version: 3.7.3
```

Push your *encrypted* secret to flux, and you should see the secret available in your cluster! Getting this workflow set up before anything else will save you some trouble when you need it.

## Installing a Helm Release

Flux also supports helm releases. This is particularly useful to me, because I find it can be easy to lose track of the original install commands and `values.yml` files that I used when maintaining a helm release. By declaring the helmrelease in gitops, there's never any confusion. Flux also will also conveniently keep your release up to date if you don't pin the version. 

The `HelmRepository` kind keeps the helm repo updated (on a 1hr interval in this case.)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: longhorn
  namespace: flux-system
spec:
  interval: 60m
  url: https://charts.longhorn.io
```

The `HelmRelease` references the above `HelmRepository`. You can also declare any helm values under `spec.values`. If i were to set the version as `>=1.3.0`, the release would be updated as new versions become available. In this case however, I'd rather pin the version and manually update the release. 

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: longhorn
  namespace: longhorn-system
spec:
  interval: 60m
  chart:
    spec:
      chart: longhorn
      version: '1.3.0'
      sourceRef:
        kind: HelmRepository
        name: longhorn
        namespace: flux-system
      interval: 30m
```

That's the most basic case, but you can also use sources of kind `GitRepository` as the source of a helm chart. The the [flux docs](https://fluxcd.io/flux/guides/helmreleases/#git-repository) for more details. 


## Defining Dependencies
Getting my flux repository set up was smooth and exciting, until I hit this major road bump. Flux doesn't know which of your kubernetes objects depend on one another by default. For instance, I want my load balancer, metallb, to be up and running before setting up my ingress controller traefik. If flux tries to reconcile traefik before metallb, that's a problem. I found myself pretty frustrated by these kinds of issues, and didn't find the documentation particularly helpful as a novice flux user. 

I ended up snooping through other flux repositories on the [awesome-home-kubernetes](https://github.com/k8s-at-home/awesome-home-kubernetes) repository list for inspiration. These repositories are extremely helpful when looking for practical implementation "advice". Several of the flux examples I found use a folder structure something like this:

```
base/
  - apps.yaml
  - core.yaml
  - crds.yaml
apps/
core/
crds/
```

You can then create `kustomizations`, defining the dependencies between these directories for flux. 

```yaml
---
# crds.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: crds
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./cluster/crds
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system

---
# core.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: core
  namespace: flux-system
spec:
  interval: 5m0s
  dependsOn:
    - name: crds
  path: ./cluster/core
  prune: false
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
---
# apps.yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 5m0s
  dependsOn:
    - name: core
  path: ./cluster/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

`specs.dependsOn` lets flux know the correct order for reconciliation. You can customize this basic idea for you needs, or jack the structure from [my repository](https://github.com/dvignoles/pikluster) if you like!

### Custom Resource Definitions

Many Helm Releases have the option to install custom resource types into your cluster bundled in with the application. I found out the hard way that is best practice **not** to do this when using GitOps. You can run into situations where the Flux reconciler tries to create a custom resource before the helm release is installed. That of course leads to a very frustrating error. It is worth the trouble to separately install those custom resource definitions, and make sure that happens *before* any resources which depend on them. Using the dependency structure from above, you can reconcile all of your CRDs before anything else. The manifests for CRDs can usually be found in the git repository of whatever project you are installing.

For example, here is a manifest which handles installing the traefik CRDs from their git repository.
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: traefik-crd-source
  namespace: flux-system
spec:
  interval: 30m
  url: https://github.com/traefik/traefik-helm-chart.git
  ref:
    tag: v10.24.0
  ignore: |
    # exclude all but the CDRs
    /*
    # path to crds
    !/traefik/crds/
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: traefik-crds
  namespace: flux-system
spec:
  interval: 15m
  prune: false
  sourceRef:
    kind: GitRepository
    name: traefik-crd-source
```

## Conclusion

All referenced `*.yaml` files are available in my [My flux repository](https://github.com/dvignoles/pikluster).

Thank you to all the unnamed awesome-home-kubernetes contributors whom I have shamelessly copied from, I could not have gotten anything working without you!