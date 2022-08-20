---
title: "Flux Secret Management with SOPS age"
date: 2021-10-23T16:16:41Z

categories:
  - Kubernetes
  - Self Hosted
tags:
  - age
  - flux
  - gitops
  - kubernetes
  - linux
  - secret
  - server
  - sops
toc: false
author: budimanjojo
slug: flux-secret-management-with-sops-age
---
![flux secret with sops age](/images/flux-secret-management-with-sops-age_1.png)

Following my [previous post](https://budimanjojo.com/2021/10/20/how-i-manage-my-kubernetes-manifests-using-flux/) about Kubernetes management using [Flux](https://fluxcd.io/), this post will cover secret management using [SOPS](https://github.com/mozilla/sops) and [age](https://github.com/FiloSottile/age).
SOPS is a tool to encrypt and decrypt text files while age is the encryption tool that is simple, modern, and secure.
<!--more-->

## Reasons to Use SOPS and age

Flux supports [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) and SOPS as secret management, but I prefer SOPS for multiple reasons.
You may or may not agree with me.
But this is not about which one is a better tool, instead this is simply a post about me sharing my thought.
The main difference between SOPS and Sealed Secrets is SOPS is not focusing mainly in Kubernetes while Sealed Secrets is.
While this can be a pro for Sealed Secrets, I prefer SOPS because I also use SOPS to manage my Ansible secrets with the [community collection](https://github.com/ansible-collections/community.sops).
Another thing I like about SOPS is it’s not doing encryption by itself and allows us as the user to use our preferred encryption tool.
I choose age as my encryption tool because it’s simple and quick.

While I love and use SOPS, there is something concerning about its development. It has been slow and it has been [looking for new maintainers](https://github.com/mozilla/sops/discussions/927) for quite some time now.
I don’t know how long this project will work in the future and I really hope they can sort it out.

## Installing SOPS and age

The first step is of course installing the tool.
In the machine that you run Flux, install SOPS and age.
You can get the binary for SOPS by grabbing it from their [Github release page](https://github.com/mozilla/sops/releases) like this:

```
sudo curl -L https://github.com/mozilla/sops/releases/download/v3.7.1/sops-v3.7.1.linux -o /usr/local/bin/sops && sudo chmod +x /usr/local/bin/sops
```

To install age, you can either use your distribution package manager or you can also grab it from their [Github release page](https://github.com/FiloSottile/age/releases) too:

```
curl -LO https://github.com/FiloSottile/age/releases/download/v1.0.0/age-v1.0.0-linux-amd64.tar.gz && sudo tar -C /usr/local/bin --strip-components 1 -xf <meta content="text/html; charset=utf-8" http-equiv="content-type"></meta>age-v1.0.0-linux-amd64.tar.gz && rm <meta content="text/html; charset=utf-8" http-equiv="content-type"></meta>age-v1.0.0-linux-amd64.tar.gz
```

## Creating age Keypair

SOPS will try to find age keypair in `$XDG_CONFIG_HOME/sops/age/keys.txt` for Linux, so let’s create our first age key there:

```
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

## Configuring SOPS

After creating our age keypair, we need to configure SOPS so that it knows what to encrypt.
In a Kubernetes secret manifest, everything below the level of data/stringData in a Secret kind are our secrets.

To let SOPS know what to look for, we can create a SOPS config file in our git repository root so we can easily use `sops` command to encrypt our secret file without typing the whole command.
Please note that this step is optional but highly recommended. This is how mine looks like:

```
$ cat ./home-cluster/.sops.yaml
creation_rules:
- encrypted_regex: '^(data|stringData)$'
  path_regex: '.*\/cluster\/.*\.yaml$'
  age: >-
    age1zeqkpfz7e3s207ynea0z0auc0mrct0pc7w4sh6j3d0c4qac3dahqj9ufdg
```

Like usual, don’t copy and paste them into your git repository, because it won’t work.
You need to match it with your own setup:

1. My local cloned git repository is inside `./home-cluster` and `.sops.yaml` is the file where SOPS will find matches.
2. Inside that file, we have `creation_rules` list:
    - encrypted\_regex tells SOPS to encrypt values under the matching regex. Here, I tell SOPS to look for key that starts and ends with `data/stringData` using regex
    - path\_regex tells SOPS what file to match this rule. You can see that I write everything inside `./cluster` directory that have `.yaml` extension using regex.
    - age is the tool to encrypt/decrypt the strings, you will need to put your age public key here (get it from your `~/.config/sops/age/keys.txt` file.

You can try encrypting your first `secret.yaml` file inline using this command (that file needs to be below or the same level as your `.sops.yaml` file):

```
sops -e -i secret.yaml
```

## Connecting SOPS to Flux

We need in-cluster secret so Flux can decrypt our secret inside our Kubernetes cluster.
Flux will look for a secret with `.agekey` as the key name.
Let’s create it:

```
kubectl -n flux-system create secret generic sops-age --from-file=keys.agekey=~/.config/sops/age/keys.txt
```

Now, we can tell Flux to use SOPS as our secret decryption tool.
You can update your Flux Kustomization (CRD) using `flux create kustomization` command or simply push the changes into your remote git repository.
Here’s how mine looks like:

```
$ cat ./cluster/base/flux-system/gotk-sync.yaml
...
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./cluster/base
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  validation: client
  decryption:            <===
    provider: sops       <===
    secretRef:           <===
      name: sops-age     <===
```

Simply add the lines highlighted with arrow in Flux kustomization custom resource where you want to use SOPS to decrypt your secrets.

## Ending

Thank you again for reading my post.
This is how I use SOPS with age as my Flux GitOps secret management.
Please leave a comment if you have any suggestion or feedback.
I would be very happy to accept any praise or critic from you!
