---
title: "How I Write Kubernetes Manifests"
date: 2023-02-22T18:45:56Z

categories:
  - Kubernetes
  - Linux
  - Terminal stuffs
tags:
  - kubernetes
  - neovim
  - vim
  - editor
  - linux
  - terminal
toc: false
author: budimanjojo
slug: how-i-write-kubernetes-manifests
---

While tools like [Helm](https://helm.sh) and [kustomize](https://kustomize.io) can significantly reduce the amount of manual [Kubernetes manifests](https://medium.com/@sujithabdulrahim/understanding-the-kubernetes-manifest-e96d680f2a11) writing, it's often impossible to completely avoid it (even (`kustomization.yaml` file is itself a Kubernetes manifest).
For instance, you may need to create a basic ingress because the chart you're using doesn't provide a template for it, or generate a certificate for your domain with cert-manager.
In this post, I'll describe how I leverage [VSCode snippets](https://code.visualstudio.com/docs/editor/userdefinedsnippets) and [yaml-language-server](https://github.com/redhat-developer/yaml-language-server) to write Kubernetes manifests.
<!--more-->

## Background

As someone who [uses FluxCD to manage my cluster](https://budimanjojo.com/2021/10/20/manage-kubernetes-manifests-using-flux/), I find myself frequently writing "Kustomization/Fluxtomization" files in my [repository](https://github.com/budimanjojo/home-cluster).
However, copying, pasting, and editing these files has become a tedious part of my routine.

To simplify this process, I rely on `yaml-language-server` to enable autocompletion.
To achieve this, I currently set the `yaml.schemas` in `yaml-language-server` by the filename, i.e. kustomization.yaml will use the schema of Kubernetes.
While this method works well for named files, it can become problematic when working with new files or those that are not named according to convention.

I'm actively seeking a more effective way to streamline this process.

## Inlined schemas to the rescue

My initial idea is to use an [inlined schema](https://github.com/redhat-developer/yaml-language-server#using-inlined-schema) instead of relying on file names to enable autocompletion.
This approach represents a significant improvement over the current method of saving and renaming files to obtain autocompletion.
For instance, here's an example of an inlined schema for a Kustomization manifest:

```
# yaml-language-server: $schema=https://raw.githubusercontent.com/SchemaStore/schemastore/master/src/schemas/json/kustomization.json
```

However, one downside of using inlined schemas is that it's not possible to use just `kubernetes` instead of the full URL for the value.
I'm considering creating a new issue for this, but it's not a major concern for me as I won't be manually writing this line.
Keep reading to find out how I automate this process.

## Writing the long URL is not an improvement

Using inlined schemas can create a new problem: it's not only difficult to remember the long lines but also highly inefficient.
To address this issue, I created snippets for the manifests I frequently write.

Here's an example workflow for writing a Kustomization file using my new approach:

1. Open a new file and type `kks` and a snippet will appear in my autocompletion list.
2. Selecting that snippet will expand this for me (`[default]` is a snippet jumpnode and `|` is where the cursor will endup at the end):
```yaml
---
# yaml-language-server: $schema=https://raw.githubusercontent.com/SchemaStore/schemastore/master/src/schemas/json/kustomization.json
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: [default]
resources:
  - |
```

Using snippets, I can avoid manually typing the `apiVersion` and `kind` keys that are required anyway, thus saving time and effort.

## This doesn't work on CRDs that I don't have schemas for

Although tools like Helm can be helpful in minimizing the need to write Kubernetes manifests manually, you'll still likely write more custom resource definitions (CRDs) than built-in API objects.
If you're using Flux, you may only need a repository [that pulls schemas provided upstream](https://github.com/JJGadgets/flux2-schemas).
However, this doesn't work for all CRDs.

Thankfully, I found a valuable resource in [@onedr0p](https://github.com/onedr0p/home-ops)'s repository.
The repository includes a GitHub workflow that uses [crd-extractor](https://github.com/datreeio/CRDs-catalog) to create schemas from your cluster.
The workflow also publishes the schema as a container with an nginx webserver, which can be deployed in your cluster.

This approach has several benefits, including hosting only the necessary schemas and having the files locally available in your homelab, which improves fetch time.
The downside is that you need a self-hosted GitHub runner to achieve this, unless you want to expose your Kubernetes cluster, which is not recommended.

## I still need to write the snippets

To streamline this process, I created a VSCode snippets compatible [repository](https://github.com/budimanjojo/k8s-snippets) containing snippets for Kubernetes API manifests and CRDs that I'm using in my cluster.
The repository is free for you to fork, pull requests, and any other usage (and stars are always appreciated!).

## Ending

Thank you for reading about my approach to writing Kubernetes manifests with the help of snippets and `yaml-language-server`.
I hope this has been helpful to you.
I'm always interested in learning new techniques and tools, so please share your own methods in the comments below.
By collaborating and sharing our experiences, we can all become more efficient and effective Kubernetes developers.
