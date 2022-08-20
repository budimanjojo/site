---
title: "Raspberry Pi Media Server Series 3 - External Drives"
date: 2019-12-09T17:35:52Z

categories:
  - Linux
  - Raspberry Pi
tags:
  - btrfs
  - external drives
  - raid
  - raspberry pi
toc: false
author: budimanjojo
slug: raspberry-pi-media-server-series-3-external-drives
---
This is the third part of Raspberry Pi Media Server series, if you haven’t read our previous part, here’s where we discussed [securing SSH server](https://budimanjojo.com/2019/11/30/raspberry-pi-media-server-series-2-secure-ssh/) on Raspberry Pi.
In this post, I’m going to share how I setup external drives on my Raspberry Pi.
Here’s a glimpse on my setup before you decided to read this entire lengthy page:
<!--more-->

- Two hard drives in [BTRF RAID1](https://btrfs.wiki.kernel.org/index.php/Using_Btrfs_with_Multiple_Devices) format for data redundancy
- Mounted on /mnt and not hanging when the drives are not connected
- Monthly auto scrub to repair corrupted data

## Getting Started

Now, let’s prepare the ingredients.
In my setup, I have a two 1TB hard drives inside a enclosure bay from Orico that looks like this:

![orico 9528U3](https://budimanjojo.com/wp-content/uploads/2019/12/4817_P_1437500396503.png)

They are connected to my Raspberry Pi 4, all the time unless the hard drives or the bay died I think.
The enclosure is powered using it’s own power adapter so I don’t have to worry about my Pi can’t give it enough juice.
Of course this is just my setup, you can have something else.
Maybe you don’t need two drives, then go with one.
Maybe you want to use SSD instead, go with it.
Or even go crazy like 5 drives RAID 10 setup, it’s up to you.

The point is, we will be using BTRFS instead of the standard EXT4 filesystem.
We are using BTRFS because it’s the closest one to [ZFS](https://en.wikipedia.org/wiki/ZFS) we have for average consumer that doesn’t have ton s of RAM.
So the conclusion of this segment is, you need to have one or more drives connected to you Raspberry Pi.
It can flash drive, HDD, SSD or anything else that can store data.

## Installing Required Programs

If you are planning on using BTRFS for the filesystem like I do, you need to install `btrfs-progs`.
Here’s the command on Manjaro Linux:

```
sudo pacman -S btrfs-progs
```

If you are on Raspbian, then the command will be:

```
sudo apt install btrfs-progs
```

After installation, you need to restart your Raspberry Pi to load the module into the kernel.
Simply do a:

```
sudo reboot
```

## Creating BTRFS File System

If you have connected your external hard drives into Raspberry Pi.
In the terminal, type in:

```
sudo lsblk
```

The command above will show you the drives connected to your Raspberry Pi.
Here’s how mine looks like:

```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    0  1.8T  0 disk
sdb           8:16   0  1.8T  0 disk
mmcblk0     179:0    0 14.9G  0 disk
├─mmcblk0p1 179:1    0   94M  0 part /boot
└─mmcblk0p2 179:2    0 14.8G  0 part /
```

Now we know what are the drive partition names that we are targeting.
On my example above, I should be targeting `/dev/sda` and `/dev/sdb`.
To create BTRFS filesystem on an un-formatted drive, you do this:

```
mkfs.btrfs /dev/partition
```

Remember to replace /dev/partition with the drive partition name, for example /dev/sda.
But if you have two or more drives like I do, we should create BTRFS with [RAID](https://btrfs.wiki.kernel.org/index.php/Using_Btrfs_with_Multiple_Devices).
For example, here’s how I create a RAID1 on my two external drives for Raspberry Pi:

```
mkfs.btrfs -d raid1 -m raid1 /dev/sda /dev/sdb
```

That’s it.
You can now proceed to mounting the external drives into you Raspberry Pi.

## Mounting External Drives

Now, we can mount the external drives into the Raspberry Pi by using this command:

```
sudo mount /dev/partition /mountpoint
```

Remember to replace /dev/partition and /mountpoint.
For example, my RAID1 external drives are located in /dev/sda and /dev/sdb.
And I want to mount them in my /mnt directory, so my command will be

```
sudo mount /dev/sda /mnt
```

I can mount either /dev/sda or /dev/sdb because they are literally one big partition in a RAID1 already.

This mount will not last on reboot though.
So now we will tell the kernel to mount our external drives on boot.
To do this, we need to edit our `/etc/fstab` file.
Here’s how it should looks like:

```
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
/dev/mmcblk0p1                             /boot vfat  defaults        0 1
/dev/mmcblk0p2                             /     ext4  defaults        0 1
UUID=0f65665c-efa5-412a-9f18-4a589f1bf9f5  /mnt  btrfs defaults,nofail 0 0
```

You can see that I’m not using `/dev/sda` or `/dev/sda` because they are not consistent on reboot.
So instead, I use the device [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier).
To get the UUID of the filesystem, type in:

```
sudo blkid
```

You can also see that I add `nofail` in the options tab.
Because we don’t want our Raspberry Pi not booting up when the external drives are not connected.
You can test your new /etc/fstab configuration by typing this command on the terminal:

```
sudo mount -a
```

If you don’t see any error then you are done.
Good job!

## Ending

So, we have our external drives mounted on Raspberry Pi.
We can put a lot of data without worrying too much about the disk space now.
If you use RAID setup with multiple drives and redundancy like I do, you also have a safe precaution if one of the drive fails.
Next, we will be setting up docker for our Raspberry Pi Media Server.
Stay tuned! 🙂
