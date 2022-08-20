---
title: "Raspberry Pi Media Server Series 1 - Installing Manjaro"
date: 2019-11-25T00:04:50Z

categories:
  - Linux
  - Raspberry Pi
tags:
  - arm
  - linux
  - manjaro
  - raspberry pi
toc: false
author: budimanjojo
slug: raspberry-pi-media-server-series-1-installing-manjaro
---
So you bought a Raspberry Pi and you don’t know what should you do with it.
Well, in this series I want to share my way of using my Raspberry Pi.
I bet you already guessed it right by reading the title.
Yes, this is the first part of my Raspberry Pi Media Server series.
Before I continue, here’s what I have for this project:

- A full set of [2GB Raspberry Pi 4 Model B](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/)
- [ORICO 3.5 inch Dual Bay Aluminum Alloy USB3.0 Hard Drive Enclosure (9528U3)](http://my.orico.cc/goods.php?id=4817)
- Two 1TB Hard Disk Drive
- A memory card reader
<!--more-->

To let you visualize what you will have by following this series, here’s a glimpse of preview for you.
You will have a media server that can download and stream media whether you are in your local network or outside.
All of this are done simply with [Docker](https://www.docker.com/) containers and [Traefik](https://traefik.io/) as the reverse proxy combined with optional [BTRFS](https://btrfs.wiki.kernel.org/index.php/Main_Page) RAID 1 for redundancy.

Now, let’s get started with installing your Raspberry Pi OS of choice.
I use Manjaro Linux on my primary computer for more than 4 years, so when I found out they released [Manjaro ARM 19.10](https://forum.manjaro.org/t/manjaro-arm-19-10-released/107677) with headless setup support, I immediately made the switch from Raspbian.
Of course, the operating system you choose is not really important, because we will be using Docker containerization for everything here.
As long as you can install OpenSSH and Docker in that OS, you are good to go.

## Execution

Here’s how you install Manjaro ARM Minimal Editiion on your Raspberry Pi 4, which should be almost the same if you want to go with Raspbian.

1. Grab the current Manjaro ARM Minimal release for Raspberry Pi 4 [here](https://manjaro.org/download/arm/raspberry-pi-4/arm8-raspberry-pi-4-minimal/).

![download manjaro arm image](https://budimanjojo.com/wp-content/uploads/2019/11/download-manjaro-arm-1024x499.png)

2. Also grab and install [balenaEtcher](https://www.balena.io/etcher/) into your current computer.

![](https://budimanjojo.com/wp-content/uploads/2019/11/download-balenaetcher-1-1024x500.png)

3. Put in your SD card into your memory card reader and plug it into your computer.
4. Open balenaEtcher program.
Select the Manjaro ARM image that you have downloaded in the “Select Image” option and select you SD card in the ” Select Drive” option.
Then click the “Flash” option.

![Flash Manjaro to SD Card with balenaEtcher](https://budimanjojo.com/wp-content/uploads/2019/11/Screenshot-from-2019-11-24-15-20-44-1.png)

5. Wait until it finished flashing and get your SD card out of the card reader.
And put it back into your Raspberry Pi 4.
6. Connect Raspberry Pi 4 to the power adapter and router (that’s it if you’re going for headless installation like me).
You may need to connect keyboard and monitor to your Pi if you don’t want to go with headless setup.
7. If you choose headless install, you need to install SSH in your current computer.
You will also need to know your Raspberry Pi’s IP address, which you can get by looking into your router’s DHCP client lists page (which is different on every router) or you can use tools like [Angry IP Scanner](https://angryip.org/) in your current computer.
If you are on Linux, simply install nmap and type in `nmap -sn <gateway-ip>/24` in your terminal will show it for you, of course replace gateway-ip with your router ip address like 192.168.1.1.
8. SSH into your Raspberry Pi by typing `ssh root@<ip>`, replace &lt;ip&gt; with Raspberry Pi’s IP address that you get from the previous step if you are on headless setup and you should get the OEM Setup.
If you are not on headless setup, you should see the OEM Setup on monitor hooked up to your Pi.
9. Follow through the OEM Setup and your Raspberry Pi will reboot after you are finished.
After reboot, `ssh root@<ip>` will no longer work, you should use `ssh <username>@<ip>` instead and it will ask for your password.
You get your username and password from the OEM Setup you’ve gone through.

So that’s it for this first part of Raspberry Pi Media Server series.
In the next part, I will be showing you how to [secure your Raspberry Pi installation](https://budimanjojo.com/2019/11/30/raspberry-pi-media-server-series-2-secure-ssh/).
Till we meet again! 🙂
