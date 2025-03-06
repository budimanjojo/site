---
title: "The Pain and Peculiarity of NixOS Networking with systemd-networkd"
date: 2025-03-06T12:20:51+07:00

categories:
  - Linux
  - Networking
  - Systemd
tags:
  - networkd
  - nixos
  - networking
  - systemd
toc: false
author: budimanjojo
slug: systemd-networkd-nixos-pain
---

Setting up networking on NixOS with `systemd-networkd` should, in theory, be a straightforward and declarative process.
Just define your network configurations in Nix, let `systemd-networkd` handle the interfaces, and enjoy a seamless, reproducible setup.
But reality hits differently.
My journey with `systemd-networkd` on NixOS has been a mix of frustration, debugging marathons, and the occasional victory that feels more like luck than mastery.
In this article, I'll go over some of the quirks I've encountered, the pitfalls that made me question my sanity, and the workarounds that (sometimes) saved the day.
<!--more-->

## Background

I migrated my homelab firewall/router from VyOS to NixOS, mainly because of [VyOS's stance on community access to LTS builds](https://blog.vyos.io/community-contributors-userbase-and-lts-builds).
While I know NixOS shouldn't be the first choice for a firewall/router, nor something I'd generally recommend, it was the best fit for me.
Most alternatives were GUI-based, non-declarative, or not Linux powered, which didn't align with what I wanted.

Since this is just for my home setup, I figured I could make it work.
That said, I wouldn't advice anyone to follow my path unless they fully understand the risks and know what they're doing.

Fast forward a few months, and my setup has been running smoothly.
I even recently deployed another NixOS machine on an Oracle Always Free instance to act as a WireGuard relay, giving me more stable remote access to my home network.

For networking, I went with `systemd-networkd` -- the approach recommended by the NixOS wiki.
And, as you might guess from the title and introduction, that decision came with its fair share of quirks.
So, let's dive into them one by one.

### 1. It's not declarative

NixOS is marketed as a declarative operating system.
Over the years of using NixOS modules, I've found that to be mostly true, when something isn't fully declarative, there's usually a warning or an explicit switch for it.
Given that `systemd` is a first-class citizen in NixOS, I was shocked to find that `systemd-networkd` doesn't follow this principle.

By "not declarative", I mean that if I tell it to create an interface, it should create it (and it does).
But if I remove that interface from my configuration, it should be deleted automatically.
Surprisingly, it isn't.

Here are two frustrating examples I encountered (you can try it yourself):

* **Lingering WireGuard interface**: While transitioning my firewall from acting as a WireGuard "relay" server to as a "peer", I temporarily added a `wg1` interface to my `systemd-networkd` config instead of modifying `wg0` directly.
Once everything was working, I removed `wg1` from my NixOS configuration.
But instead of disappearing, the `wg1` interface was stuck around, completely locking me out of my network until I manually deleted it.

* **Stale IP Masquerading rules**: I had `IPMasquerade=true` set on one of my network interfaces.
This automatically created a bunch of things, like `nftables` rules and `sysctl` settings.
Later, I decided to disable `IPMasquerade` and configure everything manually.
However, `systemd-networkd` didn't clean up its changes, causing conflicts with my manual setup.
Restarting `systemd-networkd` or running `networkctl reconfigure` did nothing.
The only way to fully reset the system was a complete reboot.

I know I'm being harsh on `systemd-networkd`, after all, it never claims to be declarative.
But this article is about using `systemd-networkd` with NixOS, where I expect tools to work in a declarative way.
Even outside of NixOS, I'd argue that `systemd-networkd` should clean up after itself rather than leaving behind orphaned interfaces and broken configurations.

NixOS usually handle these things well.
For example, when using `config.networking.interfaces`, NixOS ensures that missing interfaces are properly removed in the activation script.
So why doesn't `config.systemd.network` follow the same approach?

### 2. It messes with other tools

I run `frr` on my firewall machine to manage BGP routes for my Kubernetes cluster's load balancer.
Everything was working fine -- until one day, I suddenly lost access to my Kubernetes services exposed via the load balancer.

When I `ssh`-ed into the server and checked the `frr` logs, I found this concerning message:

```bash
Kernel deleted a nexthop group with ID (54) that we are still using for a route, sending it back down
```

A simple restart of the `frr` service restored everything.
But the same issue happened again the following week.
That's when I decided to dig deeper.

After some investigation, I discovered that `systemd-networkd` [was the culprit](https://github.com/systemd/systemd/issues/29034).
Every week, my server runs a scheduled `nixos-rebuild`, which restarts `systemd-networkd`.
And when it restarts, it silently deletes nexthops that it didn't create itself on interfaces it manages.

The fix?
Setting `ManageForeignNextHops=false`.
I also learned that `systemd-networkd` applies the same behavior to routes and routing policies -- it will delete anything it doesn't explicitly manage by default.

This kind of behavior makes it tricky to use `systemd-networkd` alongside other networking tools.
If you're using BGP, custom routes, or policy-based routing, be aware that `systemd-networkd` might just decide to wipe them out.

### 3. The strictly typed NixOS module struggle

When I tried to set `ManageForeignNextHops=false` in `networkd.conf` using the NixOS module, I naturally added this to my configuration:

```nix
config.systemd.network.config.networkConfig.ManageForeignNextHops = false;
```

But when I ran `nixos-rebuild`, it failed -- turns out, this settings wasn't recognized.

`ManageForeignNextHops` was introduced in `systemd` 256, which is relatively new, but it's already the default version in the latest stable NixOS (24.11).
Since the option wasn't available in the module, I created a PR to add it ([NixOS/nixpkgs#376630](https://github.com/NixOS/nixpkgs/pull/376630)).

That was over a month ago, and it's still not merged.
Even if it does get merged, there's no guarantee it'll be backported to 24.11.

So, my only choice was to work around it manually.
If you're facing the same issue, here's what I ended up doing:

```nix
config.environment.etc."systemd/networkd.conf.d/frr.conf".text = ''
  [Network]
  ManageForeignNextHops=false
'';
```

This creates a drop-in config for `networkd.conf` and works fine.

This isn't an isolated issue -- it happens frequently with NixOS module due to their strictly typed nature.
While I appreciate the type safety and declarative design, the way nixpkgs handles PRs is frustrating.

Right now, getting a PR merged feels like a popularity contest.
You need to "find attention" -- add a 👍 emoji, get someone to review and approve it, and hope a committer notices.
Unlike nixpkgs packages, the NixOS module system has no assigned maintainers, so I had no idea who to tag.
I ended up tagging the last person who touched the file, and ... nothing.
No response.

I know this isn't strictly a `systemd-networkd` issue, but I'm including it because anyone using `systemd-networkd` with `frr` might run into the same roadblock.

## Conclusion

These are the main quirks I've encountered while using `systemd-networkd` on NixOS.
There are plenty of smaller frustrations I didn't include -- like the general complexity of `systemd-networkd`.
It tries to do a lot, but its behavior is often inconsistent.

For example, I can understand why it doesn't automatically delete interfaces it no longer manages.
It's probably to avoid breaking things if the interface is still in use.
But then why does it aggressively removes nexthops, routes, and routing policies it didn't create by default?
The logic feels contradictory.

That said, despite all these quirks, I still like `systemd-networkd` and NixOS.
At the end of the day, I have a fully functional open-source router running exactly the way I want.
And really, what more

These are all the things that I can think of right now that I consider quirks.
There are more trivial issues that I ran into that I don't include here, like the complexity of `systemd-networkd`.
It's trying to do a lot of things, but very inconsistent in doing it.

For example, I believe the reason of it not deleting interfaces that it doesn't manage anymore is to not break things in case you are still using the interface.
This is a fair enough reason, but why does it deletes nexthops, routes, routing policies it doesn't by default?

After all this, I still like `systemd-networkd` and NixOS.
I have a working router that is using opensource tools thanks to them, what more can I ask for?
