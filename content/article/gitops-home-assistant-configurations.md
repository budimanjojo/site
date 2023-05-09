---
title: "How I GitOps Home Assistant Configurations"
date: 2021-11-04T17:37:02Z

categories:
  - Kubernetes
  - Linux
  - Self Hosted
tags:
  - gitops
  - homeassistant
  - kubernetes
  - server
toc: false
author: budimanjojo
slug: gitops-home-assistant-configurations
---
![home assistant gitops](/images/gitops-home-assistant-configurations_1.png)

I always love the idea of GitOps, where everything I have in a git repo represents the current state of my application.
But not everything are made for GitOps, so we have to sort of “make it work”.
In this post, I will show you how I manage to GitOps my Home Assistant configurations.
Spoiler, this is hacky and messy at the same time, so please bear this in mind before continuing.
<!--more-->

## My Setup

I deployed my Home Assistant in a Kubernetes cluster that I [manage using Flux](https://budimanjojo.com/2021/10/20/how-i-manage-my-kubernetes-manifests-using-flux/).
The `/config` directory is stored in a [Rook Ceph](https://rook.io/) provisioned PVC.
The `yaml` files can be seen [here](https://github.com/budimanjojo/home-cluster/tree/main/cluster/apps/default/homeassistant).
Another thing to keep in mind is that I use the official Home Assistant container image which has `git` installed in the container by default.
This is important because you will need that command in your container.

## Concept

Actually this wasn’t what I have in mind before.
I was thinking about using a [Git hooks](https://git-scm.com/docs/githooks) that will push the changes into my Home Assistant and do a restart call of my HAss deployment using `kubectl`.
This was the only way I have in mind to achieve this, and it looks weird.
Until [alex](https://github.com/alexwaibel) told me how he did it in [k8s-at-home](https://discord.gg/k8s-at-home) Discord channel (this is a great place to discuss everything Kubernetes, come join us!).

So here’s how I ended up doing.
Instead of pushing the changes into the container, I make an automation in Home Assistant to pull the changes instead.
This can be achieve using the [webhook](https://www.home-assistant.io/docs/automation/trigger/#webhook-trigger) automation trigger.
You might ask what trigger the webhook to Home Assistant, and the answer is Github.
Using this awesome [Github Action](https://github.com/marketplace/actions/webhook-action), I can send a webhook to Home Assistant whenever I push a commit into Github.

## Detailed Steps

Before doing anything, I “exec” into the Home Assistant container and setup the remote repo in the `/config` directory.
After that, everything else should be done outside of the container.
In my repository, I have this workflow:

```
name: Deploy Config

on:
  push:
    branches: [ '*' ]

jobs:
  deploy_config:
    runs-on: ubuntu-latest
    steps:
      - uses: joelwmale/webhook-action@2.1.0
        with:
          url: ${{ secrets.HASS_WEBHOOK_URL }}/update_config
          body: '{}'
```

The url is a [Github secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets) which should be `http(s)://<your-hass-domain>/api/webhook/<webhook_id>`, notice that I have `update_config` as the webhook id.
What the above workflow does is it will send a webhook to your Home Assistant whenever you do a push event to your repository.

In my Home Assistant, I create an automation like this:

```
- id: update_config_github_push_event
  alias: Update Configuration on Github Push Event

  trigger:
  - platform: webhook
    webhook_id: update_config
    local_only: false

  action:
  - service: shell_command.update_config
  - service: homeassistant.restart
```

The update\_config shell\_command is in `configuration.yaml`:

```
shell_command:
  update_config: cd /config && git pull
```

Simple, isn’t it? Everything is quite straightforward.
The automation will be triggered whenever it receives a webhook and the actions are doing a git pull and restart itself afterwards.

## Configuration Checker

You might ask, how do I make sure everything will work before pushing everything to Github and hope it doesn’t break anything?
Fortunately, there are some Github actions available for you to use that can do this.
One of them is [this](https://github.com/marketplace/actions/frenck-s-home-assistant-core-configuration-check).
But unfortunately I can’t get them working in my setup because of custom integrations that I’m using (I gave up [here](https://github.com/budimanjojo/homeassistant/commit/b8738613d20b734e492c6f767da74523008aca51)).
Basically, you will need to have a dummy `secrets.yaml` or environment variables containing the secrets you declare inside your `configuration.yaml` file.
And you will also need to make sure `custom_components` directory is tracked by your git repo, in which mine doesn’t.

## Restarts Are Annoying

Sometimes I don’t want to trigger the automation, i.e when I update a README or when there’s a typo.
Luckily this can be done without adding anything.
All you need to do is adding `[skip ci]` to your commit message you push to Github.
This tells Github to skip the workflows which will not send the webhook to Home Assistant.

## What About Secrets?

Normally in Home Assistant you declare secrets in a file called `secrets.yaml` and you call them using `!secret` in your config file.
But in a git repository you don’t want people to look at your secrets. You can either not pushing the `secrets.yaml` file entirely or you can use something like this (thanks to alex again) in your config file:

```
key: !env_var the_secret_value
```

So, instead of using `!secret`, I use `!env_var` and declare the environment variables using Kubernetes manifest.
As I’m using Flux to manage my Kubernetes secrets, I can put the encrypted secret the GitOps way which you can find [here](https://github.com/budimanjojo/home-cluster/blob/main/cluster/apps/default/homeassistant/secret.yaml).

## Closing

So this is how I GitOps my Home Assistant using a simple automation.
There are still a lot of caveats, like I can’t make sure whether the container started successfully or not.
But at least I managed to get what I wanted, which is having GitOps like experience in my Home Assistant configuration.
Please leave any idea or feedback in the comment section if you have anything to share with me.
Thank you 🙂

Updates:

- 9th May 2023: Added the required `local_only: false` for webhook automation trigger in the upcoming HAss update.
