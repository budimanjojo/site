---
title: "I Moved from Wordpress to Hugo"
date: 2022-08-20T12:14:34+07:00

categories:
  - Other
tags:
  - hugo
  - website
  - wordpress
  - static
  - cms
toc: false
author: budimanjojo
slug: moved-wordpress-to-hugo
---
![moved-wordpress-to-hugo](/images/moved-wordpress-to-hugo_1.png)

I have finally managed to get everything in this site from [Wordpress](https://wordpress.com/) to [Hugo](https://gohugo.io/).
Everything is looking great, especially for the speed that I will never get from any CMS out there.
I hesitated the move a little when people are recommending me static site generator like [jekyll](https://jekyllrb.com/) and Hugo.
I always thought that it will be hard, manually writing the entire website using `html` and `css` codes.
Turns out I was wrong all along, I haven't touch a single `html` file during this process of migrating.
<!--more-->

## Background

I'm a long time Wordpress user, but I don't really do much with my sites so I can't say I'm a power user.
When I started my first site, I was using `Blogger` (I'm really surprised that [the site is still up until now](http://budimanjojo.blogspot.com/)).

After a while, I decided to selfhost my own site using Wordpress in a cheap hosted instance.
I believe that was like 2014.
And I keep using Wordpress since that day.
I used a lot of ways to deploy Wordpress too, from using the auto install from hosting provider, to a Raspberry Pi, to using Docker container, to a Kubernetes cluster.

## Wordpress Experience

My experience on using Wordpress is quite okay.
I didn't encounter anything that break the site or anything.
My main problem with Wordpress is it's too slow even before you add any plugin to it.
And the cost of maintaining my super small site become too much, the needs of database, all the plugins that need updating, etc.

## Why Hugo

Hugo is not the only static site generator out there, the closest one is jekyll.
I think I can live with jekyll just fine, they are pretty similar.
The big selling point of Hugo for me is the single binary because it's written in Go.

## The Migration

The migration process is pretty easy.
I really love that I can run the server locally with just `hugo` command and work from there.
I use [Jekyll Exporter](https://wordpress.org/plugins/jekyll-exporter/) Wordpress plugin to get the jekyll ready zip file, then using `hugo import jekyll` command to convert it into a Hugo ready site.
From there, I installed [Bilberry Hugo theme](https://github.com/Lednerb/bilberry-hugo-theme) and use everything it provides.

## Conclusion

I'm really happy with how this goes.
I can finally gitops my entire site in a [GitHub repository](https://github.com/budimanjojo/site).
I host the entire site for free with [Cloudflare Pages](https://developers.cloudflare.com/pages/framework-guides/deploy-a-hugo-site/) which is impossible with Wordpress.
I don't need to deploy a database in my Kubernetes cluster anymore (this is the biggest reason I did this).
