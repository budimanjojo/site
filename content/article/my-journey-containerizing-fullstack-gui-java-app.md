---
title: "My Journey Containerizing Fullstack Java GUI App"
date: 2024-02-01T18:45:56Z

categories:
  - Linux
  - Docker
  - Container
tags:
  - java
  - docker
  - container
  - linux
  - x11
toc: false
author: budimanjojo
slug: my-journey-containerizing-fullstack-java-gui-app
---

Do you find yourself stuck with an outdated application, due to the absence of viable alternatives or work requirements?
Well, you're not alone.
In this blog post, I'll share my experience dealing with an old Java application that not only require an obsolete version but also demanded a specific environment setup, including a Postgres server and a CUPS server.
To liberate my system from these complexity, I embarked on the journey of containerizing the entire stack.
Join me as I recount the challenges faced and the valuable lessons learned along the way.
<!--more-->

## Overview

At my workplace, we rely on a Java accounting app, which, for undisclosed reasons, shall remain nameless as I don't think they will be happy with what I did.
Despite the app's regular updates and responsive customer support, its compatibility constraints pushed me to consider containerization.
This would not only enable universal use but also safeguard against potential abandonment, preventing me from being tied to a specific Ubuntu version.

## My Initial Plan

The program comes with a `.jar` installer file, a `jre-6u21-linux-i586.bin` and a bash script that install dependencies and Oracle JRE6 to the system.
The installer is created with [IzPack](http://izpack.org/) and I'm supposed to run the installer with the `java` binary after running the bash script.

Assuming the task would be straightforward, I began with the idea of executing the installer within a base `ubuntu:22.04` image.
Reading the official documentation fo IzPack, I tried to run the installer in unattended mode.
I created the "template" file for the installer like the path, the flavor of the app, server or client mode, etc.
However, the process turned out to be more complex due to the outdated IzPack version included not supporting unattended install, forcing me to rethink my approach.

## I Reverse Engineered Something

Frustrated by the limitations, I was about to give up.
But then I found an idea, and decided to try it out.
Knowing how IzPack works at the background, I thought about interrupting the process right after it is unpacking the bundles.
And it works!
All the required files and scripts are unpacked in the install path I specified on the installer.

Here's a little breakdown of what the installer is supposed to do:

- Copy files from `${INSTALL_PATH}/versions/${flavor}/*.jar` to `${INSTALL_PATH}/lib/` and then remove the directory.
- `cd` into `${INSTALL_PATH}/dongle` directory and run the install script then remove the directory. Seems like this is just unmodified installer they got from `SecureDongle X`.
- Update a properties file of the program to include some information needed to run the program.
- Install `postgres` 9.3 with `${INSTALL_PATH}/postgresql/postgresql-9.3.25-1-linux.run` into `/opt` directory. This installer is run with the `--superpassword` hardcoded to something I'm not supposed to know.
- Run the `${INSTALL_PATH}/postgresql/database.sh` script that will create the database template for the flavor you want to install and then delete the whole `${INSTALL_PATH/postgresql` directory.

With all this information in hand, I can just replicate the installer in a container.
This breakthrough paved the way for creating a `Dockerfile`.

I have just successfully reverse engineered the installer!

## Problems

But not without its share of challenges.
I will break them down one by one.

### Adhering to Container Best Practices

Following container best practices, I aimed for [one process per container](https://testdriven.io/tips/59de3279-4a2d-4556-9cd0-b444249ed31e/).
I should have separate container for the database and program.
However, discrepancies between server and client installations plus the need for CUPS server integration (that I just realized at this point) posed unforeseen challenges, necessitating additional troubleshooting.

I learned that the need for difference between server and client app in the installer is in the program properties file.
The server app requires `postgres-client` for the additional features like creating, backing up and restoring databases.
I can of course install only `postgresql-client` package in the program container to deal with this.
But I don't know if there's anything else that I don't know about this proprietary software.
I need time and motivation to research this up, but right now I'll just stick with including the whole `postgres` server in the server image.

There are also some other issues that I found throughout doing this.
Like I needed `cups-bsd` package alongside the minimal CUPS server for the printer to show up in the program and X11 dependencies etc.
This leads to my next problem.

### Ultra Big Image Size

The requirement of 32-bit Java version, X11 dependencies, the inclusion of CUPS and the database server inflated the image size to an undesirable 1.5GB.
Extensive modification to the `Dockerfile` like using as little layers as possible managed to reduce it to around 1.1GB for the server and 850GB for the client.

I also contributed to the increase of the size by including JRE8 instead of the provided JRE6 because this is the latest version I know this program works fine with.
JRE8 is also the latest LTS version that is still supported by Oracle.
I also installed `postgres-9.3` using the official [APT repository](https://wiki.postgresql.org/wiki/Apt) instead of the provided installer, not sure if this increase or decrease the image size.

Optimization possibilities are still being explored.

### Postgres Database Initialization at Buildtime

Usually database initialization is being done at container runtime, at least that's how I see it in other images.
I believe the reason is to have freedom in modifying username, password and the database needed when running the container.

But due to this container is supposed to be used by only this program, and the expected hardcoded password and different variant-specific SQL files by this program, I decided to initialize the database at buildtime.
This program is coded to not allowing the users to know anything about the database, I think I should respect their decision so I don't get any other problem by not following.

### Logging to STDOUT

Ideally, according to the container best practices, I should log everything to STDOUT.
But, I still don't know how to get CUPS log to STDOUT.
I don't think there's way to do this with `cupsd`.
I should be able to run a for loop to watch `/var/log/cups/error_log` file as entrypoint, but remember this is not a one process per container image?

As for the program, there is a lot of logs that I can't control.
To run the program, I use `xhost +local:${uid} && docker exec --user ${uid}` command in a `.desktop` file on the host (this is fairly easy with `home-manager` module because I use NixOS).
Running this in the terminal with `-it` flag in the `docker` command will give me logs of the program itself, but I don't do this in terminal.
And this program also logs stuffs in the `$HOME` directory of the user (I needed to create the user in the container with `useradd -m` or the program complains).

So currently, the only log that I can look at using `docker logs` command is from `postgres` which seldom or never has anything, and that's only for the server image.
I should improve this in the future, when I have the time and motivation.

### Legality to Push this Container or Show the Dockerfile

The ethical dilemma of reverse engineering a proprietary software installer raised concerns about the legality of sharing the `Dockerfile` publicly.
As a result, the project remains confined to a private GitHub repository.
Luckily, because I'm using NixOS I can easily just use the `oci-containers` module that can manage per container registry authentication which is super dope!

## Conclusion

This experience marks the first time I used `docker` build secrets by passing a `sops` encrypted build environment variable to the image safely.
Another interesting thing I found is I got the annoying "beep" sound when pressing backspace on an empty input field (you should know if you ran ancient OS before).
I don't know how it got into the program inside a container.
Is it because my USB speaker being passed to the container? Or is it because of X11 socket?
It's fascinating and annoying at the same time.

I also learned that I need to set `_JAVA_AWT_WM_NONREPARENTING=1` environment variable to the container and not the host when using some newer window managers.
Even though the display is being passthroughed to the host, but the variable only takes effect when set inside the container.

In conclusion, the journey of containerizing this ancient Java app brought forth numerous challenges and valuable insights.
From optimizing image sizes to navigating database intricacies, each hurdle presented an opportunity to learn and grow.
While some challenges remain unresolved, the experience has been instrumental in expanding my knowledge and skill set.
As the `Dockerfile` and containers remain in a private repository, I reflect on the intricate balance between open-source principles and legal considerations in the realm of software containerization.

I think I've covered everything I want to share in this post.
Thank you for reading and I hope you enjoy it as I do enjoy writing it.
