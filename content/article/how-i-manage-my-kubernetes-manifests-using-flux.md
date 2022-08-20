---
title: "How I Manage My Kubernetes Manifests Using Flux"
date: 2021-10-20T18:45:56Z

categories:
  - Kubernetes
  - Linux
  - Self Hosted
tags:
  - flux
  - gitops
  - kubernetes
  - linux
  - server
toc: false
author: budimanjojo
slug: manage-kubernetes-manifests-using-flux
---
![manage kubernetes manifests flux](/images/how-i-manage-my-kubernetes-manifests-using-flux_1.png)

GitOps is currently the most popular way to manage your services.
It’s sort of what DevOps is but with Git.
To put it in a simple way, it is a practice where you manage everything through git.
Whatever you have in your git repository is what your cluster current state is.
Because GitOps is so popular there are a lot of new tools focusing on GitOps right now, and [Flux](https://fluxcd.io/) is one of them.
In this post I will give you a glance on how I manage my Kubernetes manifests using Flux.
<!--more-->

## Prerequisite

These are basically all you need to get started:

- A Kubernetes cluster already running
- A git repository that your Kubernetes cluster can access, I use [Github](https://github.com)
- Access to your Kubernetes cluster using kubectl

## Installing Flux CLI

You will need to install Flux CLI In the machine that you can do kubectl commands into your Kubernetes cluster.
Depending on your OS, you can get it from [Homebrew](https://brew.sh/), [Chocolatey](https://chocolatey.org/), [AUR](https://aur.archlinux.org/), [GoFish](https://gofi.sh/), or you can get it from their [Github](https://github.com/fluxcd/flux/releases) repository using this one liner:

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

## Bootstrapping to Github

To connect Flux with your Github repository, [create a personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) and check all the permissions under “repo”.
Export the generated token in a environment variable named GITHUB\_TOKEN and run flux bootstrap command:

```
export GITHUB_TOKEN=<your token>
flux bootstrap github --owner your-github-username --repository repository-name --path path-to-flux --personal
```

Running the above command will create a Github repository if it doesn’t exist or push a commit if it already exists.
You can use anything you want for the “–path” flag.
Flux will create a directory called flux-system inside that path and it will contain all yaml files to get you started.
Now, you can start putting kubernetes manifest files inside the directory where you specify the “–path” at.

Let’s clone the github repository into your machine now.

```
git clone https://github.com/your-github-username/repository-name
```

## Creating Flux Kustomization

Now, whatever you put inside the directory inside the repository will be deployed into your Kubernetes cluster.
In my [repository](https://github.com/budimanjojo/home-cluster), I use `./cluster/base` as the path where I tell Flux where to look for manifests.
There are two ways to deploy services with Flux, using [kustomize](https://kustomize.io/) or [helm](https://helm.sh/).
You don’t need to have kustomize or helm installed in your machine, Flux have both helm and kustomize controllers running in your cluster already.
I don’t use Helm, so I will show you how I do this using kustomize.
I do mine using Flux CLI like this:

```
cd home-cluster
flux create kustomization apps --interval 1m --path ./cluster/apps --prune true --source GitRepository/flux-system --export > ./cluster/base/apps-kustomization.yaml
git add . && git commit "create apps kustomization" && git push
```

Please note that you don’t have to do the exact same thing as I put above, understanding about what the commands do is more important.
These are what the above commands do:

1. I change directory into home-cluster, which is my repository name. Replace it with the repository name you created in the bootstrap process.
2. I create a app-kustomization.yaml file using `flux create` CLI command:
    - the name of my kustomization is `apps`
    - –interval 1m tells Flux to to synchronizing every 1 minute
    - –path ./cluster/apps tells Flux the directory to look for manifest files
    - –prune true tells Flux to delete resources in the directory if I delete them from Github
    - –source GitRepository/flux-system tells Flux to use this repository, flux-system GitRepository kind is automatically created by Flux when we do bootstrap command. This means you can create another source from another repository if you want to. You can use `flux create source` command to do this (not covered here)
    - –export tells Flux to export the output of this command which is basically a Kubernetes manifest
    - &gt; ./cluster/base/apps-kustomization.yaml pipe the output into a file
3. I commit and push the changes into Github, so that Flux will see it.

There are a lot more available flags that you can use in flux create kustomization command that I don’t show here.
You can read them all [here](https://fluxcd.io/docs/cmd/flux_create_kustomization/).

## Deploying Your Services Using Flux

After creating Flux kustomization CRD using the step before, Flux will watch for the directory where you define with `--path` flag.
Mine is in `./cluster/apps`.
So let’s create an example service, let’s say I want to deploy [adminer](https://www.adminer.org/).

```
mkdir -p ./cluster/apps/adminer
cat ./cluster/apps/adminer/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adminer
  labels:
    app.kubernetes.io/name: adminer
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: adminer
  template:
    metadata:
      labels:
        app.kubernetes.io/name: adminer
    spec:
      containers:
        - name: adminer
          image: adminer:4.8.1
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              protocol: TCP
              containerPort: 8080
EOF
cat ./cluster/apps/adminer/kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: default
resources:
- deployment.yaml
EOF
git add . && git commit -m "deploy adminer" && git push
```

Again, you don’t have to follow whatever I do here.
This post is more about explaining how it works rather than a tutorial, hence the title is how I manage my kubernetes manifests using Flux.
What I did with the commands above are:

1. Make a directory in `./cluster/apps/adminer`.
2. Create a deployment.yaml file which contains the Deployment kind of Kubernetes manifest inside the directory created above.
3. Create a kustomization.yaml file which contains Kustomization kind of Kubernetes manifest to tell Flux kustomization controller what file to sync with my Github repository.
4. I commit and push the changes into Github, so Flux will see it.

Now, wait for 1m (depending on your –interval flag of your flux kustomization) and adminer will be deployed in your cluster.
If you are impatient like me, you can use this command to force Flux to reconcile without waiting:

```
flux reconcile kustomization apps
```

## Ending

Thank you for reading my post, I hope you can learn something from this.
Of course, there are a lot more that you can do with Flux.
It has source, kuztomize, helm, notification, image controllers that you can make use of.

This is how I manage my Kubernetes manifests using Flux GitOps, tell me more about yours in the comment section.
