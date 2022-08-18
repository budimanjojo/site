---
author: budimanjojo
categories:
- Linux
- Raspberry Pi
- Terminal Stuffs
date: "2019-11-30T10:31:59Z"
guid: https://budimanjojo.com/?p=120
id: 120
tags:
- arm ssh raspberry pi linux server
title: 'Raspberry Pi Media Server Series #2 - Secure SSH'
url: /2019/11/30/raspberry-pi-media-server-series-2-secure-ssh/
slug: raspberry-pi-media-server-series-2-secure-ssh
---
This is the second part of my Raspberry Pi Media Server series. On [previous post](https://budimanjojo.com/raspberry-pi-media-server-series-1-installing-manjaro/), I wrote about installing Manjaro Linux ARM into my Raspberry Pi 4. Now, we are going to secure our SSH server on Raspberry Pi so that we can at least prevent [brute-force](https://en.wikipedia.org/wiki/Brute-force_attack) attempt on our future home media server from the evil outside world. Securing SSH server is one of the most crucial part of every server that can be accessible on the internet.
<!--more-->

So before we begin, here’s a glance on what we are going to have by the end of this tutorial:

- Change the default port of SSH server on the Raspberry Pi
- Disabling password authentication to connect to Raspberry Pi
- Only use public key and/or two factor authentication
- Disable root login on SSH

## Getting Started

Without further ado, let’s get started. First of all, open a terminal in your Raspberry Pi. If you’re using headless setup like I do, then log into your Raspberry Pi using SSH:

```
ssh <username>@<ip>
```

Also don’t forget to do a system upgrade on your Pi:

```
sudo pacman -Syu
```

If you are using Rasbian instead of Manjaro as the operating system, then here’s the command instead:

```
sudo apt update && sudo upgrade
```

Now, follow these steps carefully because if you do anything wrong, you will risk being not able to log into your Pi using SSH anymore. That would be a disaster if you are on headless setup and not having the cable to connect your Pi to a monitor.

## Installing the Required Programs

We are using two-factor authentication method to secure our Raspberry Pi SSH server. You will need to install Google Authenticator and a text editor of your choice. I’m a Neovim guy, so here’s the command for me:

```
sudo pacman -S libpam-google-authenticator neovim
```

If you’re using Raspbian, then command will be:

```
sudo apt install libpam-google-authenticator neovim
```

## Setting Up Google Authenticator

You will need to install a generator on your smartphone for this. Currently there are [FreeOTP](https://freeotp.github.io/), [Google Authenticator](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2), [Authy](https://authy.com/). Just choose one of them. I prefer FreeOTP as it’s an opensource project but personally I use Authy because FreeOTP is not getting any update for a long time. After that, type in `google-authenticator` in your terminal and it will ask you some questions. Type y or n to answer those questions. Here’s how I answer them.

![google-authenticator-setup](https://budimanjojo.com/wp-content/uploads/2019/11/screenshot-2019-12-01_00-50-27-987x1024.png)  
As you can see, it will show you an URL link, a QR code, and a secret key. You can just scan the QR code using the app you installed on your smartphone or just type the secret key into the app. If there’s no QR code shown in the terminal, you need to install `qrencode` on your Pi or just click on the link and open it from your browser.

The app will then give you a verification code that you can put into the terminal to answer the second question.

## Setting Up SSH

Now we are going to setup our SSH server on the Pi. Using your favorite text editor, edit the file `/etc/ssh/sshd_config`. For Neovim user, the command will be

```
sudo nvim /etc/ssh/sshd_config
```

Now let’s modify something in that file. Note that when I mention “find and replace”, don’t forget to [uncomment the line](https://www.howtogeek.com/118389/how-to-comment-out-and-uncomment-lines-in-a-configuration-file/) if it’s commented out. Also, you can create a new line if you can’t find the line in that file.

1. Change the default port 22 to something else. For example 2222. Find and replace the line `Port 22` to `Port 2222`. Also, I have to mention that this part is a little bit tricky, because you can also do this on router level like I do. In my setup, I use the default port 22 on my Pi and port forward incoming port 2222 to port 22 of my Raspberry Pi instead. So that I can connect to port 22 on my LAN and only use port 2222 when connecting from outside my home only. If you don’t know what I’m talking about then you might want to just do as what I show you above.
2. Now let’s disable password authentication so we don’t need to type in a password to SSH into our Raspberry Pi. Why? Because password based authentication is not secure and people will try to brute-force random password to connect into your Pi. You will be surprised to see how many attempts I’ve seen in my router log files before I change my SSH port and disable password authentication for my internet accessible home server before. To do this, find and replace the line`PasswordAuthentication yes` to`PasswordAuthentication no`.
3. And now let’s tell SSH to use public key and/or two-factor authentication to authenticate the connection. Find and replace the line `ChallengeResponseAuthentication no` to `ChallengeResponseAuthentication yes`. Also add in this line:

```
AuthenticationMethods publickey keyboard-interactive:pam
```

if you want SSH to let you login using either public key or two-factor authentication. If you want to force SSH to let you login using both public key and two-factor authentication, use comma instead of space like this:

```
AuthenticationMethods publickey,keyboard-interactive:pam
```

4. Now you can save the file and exit your text editor.

## Setting Up PAM

So we are done with the SSH part, but not entirely done yet. We still need to tell [PAM](https://en.wikipedia.org/wiki/Linux_PAM) to use Google Authenticator instead of plain password. This is required because Google Authenticator is a PAM module. Remember that we have set SSH to use public key and/or PAM to authenticate. So now it makes sense if we set PAM to use Google Authenticator instead of plain password. To force PAM to use two-factor authentication, edit the file `/etc/pam.d/sshd` and add this line on the first line of that file:

`auth required pam_google_authenticator.so`

Save the file and exit your text editor. Here’s how my `/etc/pam.d/sshd`looks like now:

```
#%PAM-1.0
#auth     required  pam_securetty.so     #disable remote root
#auth      include   system-remote-login
auth      required  pam_google_authenticator.so
account   include   system-remote-login
password  include   system-remote-login
session   include   system-remote-login
```

## Finishing

Now, re-read this page from the beginning and check if you have done everything correctly before doing this last command. Because if you messed up something, you might be locked out of your Pi if you’re on a headless setup. If you’re sure everything is correct, then in your terminal type in:

`sudo systemctl restart sshd` to restart SSH server with the new setting.

Now, whenever you want to connect to your Pi using SSH, you will need a verification code that will be different every 30 seconds using your installed generator app in your phone. If you SSH into your Pi a lot with a computer, you can use public key for that computer by sending your public key to your Pi by typing:

```
ssh-copy-id <username>@<ip> -p 2222
```

using a terminal on your computer, type in the verification code and you will not be asked for verification code anymore if you set up SSH to use public key or two-factor authentication on the step 3 of setting up SSH.

That’s it for this part of the series. Now we are one step forward by having a more secure SSH server on our Raspberry Pi. I hope to see you again on the next part which we will be [setting up external storage](https://budimanjojo.com/raspberry-pi-media-server-series-3-external-drives/) on Raspberry Pi. 🙂
