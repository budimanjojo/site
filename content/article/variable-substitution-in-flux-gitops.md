---
title: "Variable Substitution in Flux GitOps"
date: 2021-10-27T13:36:08Z

categories:
  - Kubernetes
  - Linux
  - Self Hosted
tags:
  - cluster
  - flux
  - gitops
  - kubernetes
  - server
toc: false
author: budimanjojo
slug: variable-substitution-in-flux-gitops
---
![variable substitution in flux](/images/variable-substitution-in-flux-gitops_1.png)

Sometimes there are values that you want to use multiple times in your manifest files.
Usually, we define them using a config map or secret and we either mount them as a file or environment variable.
This is easy if you don’t use GitOps and have a small amount of pods.
Also, having a single configmap and secret will clean up a lot of mess out of your cluster, and this is what variable substitution will do for you.
I find this really useful especially for sensitive information you don’t want people all around the world to see.
In this post, I will show you how I do variable substitution using Flux GitOps tool.
<!--more-->

## Use Cases

One of the things that *triggered* me ([and many other people](https://github.com/kubernetes/kubernetes/issues/52787) ) when I tried to move from using docker swarm to kubernetes was the inability to use variable substitution in the manifest files.
Sometimes you want to share your manifest files to other people.
And of course you don’t want to leak sensitive information all over the place.
Here is a simple snippet comparing chunk of a manifest file with and without variable substitution:

*with variable substitution*:
```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
...
spec:
  routes:
    - kind: Rule
      match: Host(<span style="font-size: inherit; white-space: nowrap;">`${SECRET_HOST}.${SECRET_DOMAIN}</span><span style="font-size: inherit;">`)</span>  <===
...
```

*without variable substitution*:
```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
...
spec:
  routes:
    - kind: Rule
      match: Host(<span style="font-size: inherit; white-space: nowrap;">`mysecrethost.mysecretdomain.com}</span><span style="font-size: inherit;">`)</span>  <===
...
```

Imagine putting your secret service url into a git repository, it will be a disaster not only for your privacy but you can put your entire network at risk if someone tries to hack into it.

Another use case that I find useful is you can use configmap instead of secret for all your application configurations, for example:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-configs
data:
  grafana.ini: |
    [server]
    root_url = https://${SECRET_HOST}.${SECRET_DOMAIN}
...
```

## Implementation

Great, but how do I achieve this?
Actually it’s really simple in [Flux](https://fluxcd.io/).
All you need to do is to tell Flux about where to substitute from using the [Kustomization](https://fluxcd.io/docs/components/kustomize/kustomization/) CRD.
If you followed my previous post about [**how I deployed Flux into my Kubernetes cluster**](https://budimanjojo.com/2021/10/20/manage-kubernetes-manifests-using-flux/), you should know how to create a kustomization file using Flux CLI already.
Inside that file, you can add a `postBuild` section under `spec` and give it a ConfigMap or Secret that you want to substitute from:

```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
  dependsOn:
    - name: crds
  interval: 1m0s
  path: ./cluster/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  postBuild:  <===
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
      - kind: Secret
        name: cluster-secret-vars
```

After that, you will need to create the ConfigMap and/or Secret in the same namespace as the Kustomization kind with the name you defined:

```
apiVersion: v1
kind: Secret
metadata:
    name: cluster-secret-vars
    namespace: flux-system
type: Opaque
stringData:
    SECRET_HOST: secret_host
    SECRET_DOMAIN: secret_domain.com
```

You can either apply the manifest file without committing it into your git repository or you can use a secret management tool that Flux supports like SOPS or SealedSecret.
I wrote about how I manage my secret using SOPS [here](https://budimanjojo.com/2021/10/23/flux-secret-management-with-sops-age/) if you want to read it.

## Ending

Thank you for reading my post.
I hope this can be useful for you.
It will be great if you can leave some feedback in the comment section below.
I hope you can now convert to use GitOps style using variable substitution with Flux safely.
