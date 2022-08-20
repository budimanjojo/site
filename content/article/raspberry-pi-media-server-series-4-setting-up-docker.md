---
title: "Raspberry Pi Media Server Series 4 - Setting Up Docker"
date: 2019-12-18T01:24:55Z

categories:
  - Linux
  - Raspberry Pi
  - Self Hosted
tags:
  - arm
  - docker
  - linux
  - raspberry pi
toc: false
author: budimanjojo
slug: raspberry-pi-media-server-series-4-setting-up-docker
---
This is the forth part of my Raspberry Pi Media Server series.
If you haven’t read our previous part, [here’s](https://budimanjojo.com/2019/12/09/raspberry-pi-media-server-series-3-external-drives/) where we discussed setting up external drives on Raspberry Pi.
In this part, we are going to setting up [Docker](https://www.docker.com/) on Raspberry Pi.
Here’s what you will get if you follow through the end of the tutorial:

- Docker and Docker Compose installed
- Docker data will be in the external drives
- [Systemd](https://wiki.archlinux.org/index.php/systemd) will only start Docker if external drives are mounted

## Getting Started

Before we get our hands on setting up docker on our Raspberry Pi, let’s first do a system update.
If you are on Manjaro, the command will be:

```
sudo pacman -Syu
```

And if you are on Raspbian and Debian’s derivatives, then the command will be:

```
sudo apt update && sudo upgrade
```

You may need to reboot your Raspberry Pi if there’s a kernel or firmware upgrade.

## Installing Required Programs

The packages we need are `docker` and `docker-compose`.
To do this on Manjaro ARM, it’s really simple. Just do:

```
sudo pacman -S docker-compose
```

If you are on Raspbian and Debian’s derivatives, then here’s what you need to do (I copy pasted from [here](https://withblue.ink/2019/07/13/yes-you-can-run-docker-on-raspbian.html), thanks Alessandro Segala):

```
# Install some required packages first
sudo apt update
sudo apt install -y \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common

# Get the Docker signing key for packages
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg | sudo apt-key add -

# Add the Docker official repos
echo "deb [arch=armhf] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
     $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list

# Install Docker
# The aufs package, part of the "recommended" packages, won't install on Buster just yet, because of missing pre-compiled kernel modules.
# We can work around that issue by using "--no-install-recommends"
sudo apt update
sudo apt install -y --no-install-recommends \
    docker-ce \
    cgroupfs-mount
```

You might want to add your user to the docker group, so you don’t need to use root privileges to run docker commands.
Here’s the one liner, replace &lt;username&gt; with the user you are using on your Pi:

```
sudo usermod -aG docker <username>
```

## Using External Drives for Data

Your docker data might be getting bigger and bigger as time goes by.
That is not really a good idea to use the small and slow SD card to store them.
That is why I prefer storing them in my RAID1 BTRFS external drives which I mentioned in the [previous part](https://budimanjojo.com/raspberry-pi-media-server-series-3-external-drives/).
To do this, we will use mount binding using fstab.

Let’s create a folder you want to bind our Docker data to.
I have my external drives mounted on `/mnt`, so in my example I will create a folder `/mnt/docker-root`:

```
sudo mkdir /mnt/docker-root
```

Also, make sure `/var/lib/docker` exists in your system.
It should be there if you have started docker service (which should be automatic in Raspbian but not in Manjaro).
If the folder doesn’t exist, let’s create it and give it a right permission:

```
sudo mkdir -p /var/lib/docker && sudo chmod -R 711 /var/lib/docker
```

Now, let’s add this line into your `/etc/fstab` file (replace &lt;directory&gt; to the directory you created above, mine will be /mnt/docker-root):

```
<directory>                            /var/lib/docker  auto    defaults,bind,nofail                     0  0
```

You can now test it out by doing a `sudo mount -a` in your terminal to make sure there’s no error with your fstab file.

## Optional

You can tell systemd to only enable docker if your /var/lib/docker is mounted in your external drives.

To do this, type in:

```
sudo systemctl edit docker.service
```

This will open up a text editor of your choice, now add these lines in the file:

```
[Unit]
After=var-lib-docker.mount
Requires=var-lib-docker.mount
```

Save and exit.
That’s it.
Systemd will not start docker service if your docker data is not mounted in your external drives.

## Enable Docker

To start docker on startup, do:

```
sudo systemctl enable docker.service
```

## Ending

We have just finished setting up Docker in our Raspberry Pi.
In the next part, we will proceed on getting our hands on setting up DNS and port forwarding our Raspberry Pi so we can access it from the outside world.
Stay tuned! 🙂
