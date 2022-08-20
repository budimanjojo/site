---
title: "Automating Kubernetes Nodes Reboot with Kured"
date: 2021-10-28T16:05:01Z

categories:
  - Kubernetes
  - Linux
  - Self Hosted
tags:
  - gitops
  - kubernetes
  - kured
  - linux
toc: false
author: budimanjojo
slug:: kured-kubernetes
---
![kured-kubernetes](/images/kured-kubernetes_1.png)

Having an up-to-date Kubernetes node is important, especially security updates.
Sometimes an update requires a reboot to take effect, and you don’t want to keep checking them manually.
In this post, I will share about how I automate my Kubernetes nodes to reboot when necessary using [Kured](https://github.com/weaveworks/kured).
<!--more-->

## Linux Distro Specific

Before doing the actual setting up process, every Linux distribution has its own way to decide whether a reboot is required or not.
As I’m using Ubuntu server edition as my Kubernetes nodes distribution, I will show you how to do this in Ubuntu (it should works with Debian as well).
If you are using other server oriented distribution i.e CentOS, RHEL, OpenSUSE Leap, CoreOS, etc they will have their own way of doing automatic updates and notify you to reboot.
If you are using desktop Linux distributions like Arch Linux (which I used to do), you might need to create your own script for that.

## Enabling Unattended Upgrade in Ubuntu

Like I mentioned before, I will show you how I do it using Ubuntu as this is where my Kubernetes nodes live.
It is really simple to enable auto update in Ubuntu server.
If you have a lot of nodes to configure, doing this using [Ansible](https://www.ansible.com/) will be so much easier.

1. Make sure the required packages are installed and enabled (it’s installed and enabled by default in Ubuntu 20.04 server edition)

```
sudo apt install unattended-upgrades update-notifier-common && sudo systemctl enable --now unattended-upgrades
```

2. Configure unattended-upgrades, edit the file `/etc/apt/apt.conf.d/50unattended-upgrades`.
The default in 20.04 is already pre-configured with security updates enabled, I set mine to enable all repositories though.

```
$ cat /etc/apt/apt.conf.d/50unattended-upgrades

// Automatically upgrade packages from these (origin:archive) pairs
//
// Note that in Ubuntu security updates may pull in new dependencies
// from non-security sources (e.g. chromium). By allowing the release
// pocket these get automatically pulled in.
Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}";
        "${distro_id}:${distro_codename}-security";
        // Extended Security Maintenance; doesn't necessarily exist for
        // every release and this system may not have it installed, but if
        // available, the policy for updates is such that unattended-upgrades
        // should also install from here by default.
        "${distro_id}ESMApps:${distro_codename}-apps-security";
        "${distro_id}ESM:${distro_codename}-infra-security";
        "${distro_id}:${distro_codename}-updates";
        "${distro_id}:${distro_codename}-proposed";
        "${distro_id}:${distro_codename}-backports";
};
```

3. Lastly, edit the file `/etc/apt/apt.conf.d/20auto-upgrades` and make sure the first 2 lines below are set to “`1`” and alternatively I also added the third line to do auto-clean every 7 days.

```
$ cat /etc/apt/apt.conf.d/20auto-upgrades

APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

## Deploying Kured

Once we already have our Ubuntu node configured like above, now we have to deploy Kured in our Kubernetes cluster.
Deploying Kured can be done using their [Helm chart](https://github.com/weaveworks/kured/tree/main/charts/kured) or you can use [Flux](https://fluxcd.io/) and [Kustomize](https://kustomize.io/) like I do [here](https://github.com/budimanjojo/home-cluster/tree/main/cluster/apps/kured).
I recommend you to read my post about [how I manage my Kubernetes cluster using Flux](https://budimanjojo.com/2021/10/20/how-i-manage-my-kubernetes-manifests-using-flux/).

## Configuring Kured

This step can be done at deploy time or after deploying.
There are a lot of flags you can configure which you can look at [here](https://github.com/weaveworks/kured#configuration).
But if you are using Ubuntu like me, make sure that `--reboot-sentinel` is set to `/var/run/reboot-required`.
You can also set up notification with [shoutrrr](https://containrrr.dev/shoutrrr/v0.5/services/overview/) formatting using the `--notify-url` flag.

## Testing

Waiting for your Kubernetes node to tell Kured it’s time to reboot might take you forever.
You can test whether it’s working by triggering `--reboot-sentinel` or `--reboot-sentinel-command` flag you have set.
In Ubuntu, you can login into one of your nodes and create an empty file in `/var/run/reboot-required`:

```
sudo touch /var/run/reboot-required
```

You can set the –period to a shorter duration (default is 1 hour) to have Kured try to check whether reboot is required or not. I have mine set to `--period 5m` so I don’t have to wait for too long.

## Ending

Thank you for reading my post!
Hopefully this post can help you in some ways.
I would be very happy if you can leave me some comments below.
