---
title: "Managing Dotfiles With Chezmoi"
date: 2021-12-13T23:54:07Z

categories:
  - Desktop Stuffs
  - Linux
tags:
  - chezmoi
  - dotfiles
toc: false
author: budimanjojo
slug: managing-dotfiles-with-chezmoi
---
![managing-dotfiles-with-chezmoi](/images/managing-dotfiles-with-chezmoi_1.png)

The biggest difference between my setup and your setup might be our configuration files.
In Unix world we call them dotfiles.
Dotfiles are files whose filename starts with a dot and hidden by default in our file managers.
<!--more-->

Usually we have several machines that we use consequently, and we want all those machines to share the same environment.
And that’s why most most people will have their own collection of dotfiles stored somewhere so they can easily migrate them over to their new machines.

This could be easily achieved without having any software, you can just store them in your Github repository right?
Right, but not everything will go as you want it to be.
Here’s my take on what problem you might encounter:

- Sometimes you have different versions of programs that needs changes in the dotfiles
- You don’t want your secrets like credentials etc in a git repository unencrypted
- You need to do something on your brand new computers before you can use those dotfiles
- The path where those dotfiles should be are different across machines

You may have more problem than I can think of right now.
But in my experience, [chezmoi](https://github.com/twpayne/chezmoi) covers everything.
Please note that this is not a step by step tutorial on how to use chezmoi because it’s a very complex program and you’ll need a lot of readings.
You can start from the easy to follow [official documentation](https://github.com/twpayne/chezmoi/tree/master/docs) to get started.
This post is more about me sharing how I’m managing my dotfiles with chezmoi.

## Background

I have been using Linux as my primary OS since 2012.
I had my first dotfile like a year after using Linux when I switched to use [ZSH](https://www.zsh.org/) as my default shell.
I don’t remember how I managed them back then, but from my dotfiles [repository timeline](https://github.com/budimanjojo/dotfiles/graphs/contributors) I first created that repo in June 2014.
I started out by symlinking my dotfiles from git repository into my `$HOME` directory, to having an install script written in bash, to having [ansible-playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html), to using chezmoi.

From my experience all this years, I can feel how much chezmoi helped me in managing my dotfiles and I want to share it.

## Installing chezmoi

The first thing you need to do before starting to use chezmoi is of course installing the program itself.
There are a [lot of ways to install chezmoi](https://github.com/twpayne/chezmoi/blob/master/docs/INSTALL.md), but I like the put them in a [script](https://github.com/budimanjojo/dotfiles/blob/master/install.sh).
So I can just run one command like this to install chezmoi:

```
curl https://raw.githubusercontent.com/budimanjojo/dotfiles/master/install.sh | bash
```

Once installed, you can create a repository in Github or Gitlab or any other similar tools to host your git repository.
To simplify using chezmoi, I suggest you to name your repository `dotfiles`.
Once done, you can run `chezmoi init <your remote repository username>` and you can start adding your dotfiles with `chezmoi add` command.

## Understanding the Concept

Chezmoi is very different from other dotfiles manager tools out there, it might be confusing at first.
Especially if you used to use the traditional ways like symlinking or using [stow](https://www.gnu.org/software/stow/).

What you need to know is, try to use `chezmoi` command as much as you can.
For example, to add your [Alacritty](https://github.com/alacritty/alacritty) config, you do `chezmoi add ~/.config/alacritty/alacritty.yml`.
This will create the file in your repository with this path: `dot_config/alacritty/alacritty.yml`.
You can then do git add, commit, and push the changes to your remote repository. On your other machine, you can then install and init chezmoi, then do `chezmoi apply` to copy your Alacritty config into your new machine.

In summary, you do `chezmoi init` command once, then add files into your repository with `chezmoi add`.
Once you have everything added, in your new machine you simply do `chezmoi apply` and everything will be applied to your new machine.
Also, there is `chezmoi update` command to update your local repository to match your remote repository.
I suggest you to [setup chezmoi completion](https://www.chezmoi.io/docs/reference/#completion-shell) for your favorite shell so you can start managing dotfiles with chezmoi be easier.

## What Is the Difference From Other Tools

I can see you guys might be asking what is the difference from other tools.
You can look at this [comparison table](https://github.com/twpayne/chezmoi/blob/master/docs/COMPARISON.md#comparison-table).
For me, chezmoi have everything you may or may not have.
It’s like combination of all the other dotfiles managers out there that do everything.
Here are the features that I’m currently using intensively.

### 1. Templating

This is by far one of the features that I love so much when using chezmoi.
To add a template in chezmoi, you can use `--template` flag when doing the `add` command like this: `chezmoi add --template ~/.gitconfig`.
This command will create a `dot_gitconfig.tmpl` file in your repo that you can start templating.
What’s the use case of this?
A lot, I mean really a lot. You can use it to change a value of a dotfile according to hostname, OS, etc.

When you do chezmoi init, it will create a set of variables that you can use.
You can look at them by doing `chezmoi data` command, it’s similar to what `ansible facts` is.
You can also add your own “data” using chezmoi [config file](https://github.com/twpayne/chezmoi/blob/master/docs/REFERENCE.md#configuration-file).

I use template for things machine specific config, for example I want my terminal font to be size 12 in my main machine, but I want it to be size 14 in my work machine.
I also use it to disable something when it doesn’t match some conditions, for example I want to load `kubectl` command completion when I’m at my work machine but not on my home machine.

### 2. Ignore File

You can also add a file named `.chezmoiignore` in your repository and chezmoi will ignore the files listed inside.
It’s like `.gitignore` file, but the patterns you put inside should match the `source` file and not the `destination` file.
Usually you put your `README.md` or `LICENSE` in here so they don’t get into your `$HOME` directory.
But the great thing about this file is you can also do templating inside it.
I use this to ignore files base on conditions, for example I ignore window manager specific dotfiles when the machine is headless etc.

### 3. Scripts

Another powerful feature from chezmoi is scripting.
You can add scripts anywhere in your repository or put them in `.chezmoiscripts` directory.
The filename has to starts with `run_` and ends with `.tmpl`.
Basically these scripts will run everytime you do `chezmoi apply` command, but you can tell it to run once per machine or only run when there are changes.
You can also specify whether the scripts should be run before or after copying the dotfiles.
For example, to have the script run once before updating the dotfiles, create the filename like this: `run_once_before_<do something>.sh.tmpl`.  

There’s one thing you need to be aware of, chezmoi will run these scripts using `exec` command.
So you need to make sure you provide `<a href="https://bash.cyberciti.biz/guide/Shebang">shebang</a>` inside the file. Also, try to make those scripts as [idempotent](https://en.wikipedia.org/wiki/Idempotence) as you can so it doesn’t do the same thing over and over again whenever you run `chezmoi apply`.

I use this feature to do packages installation, setting up shell, downloading something, etc.

### 4. Encrypted Secrets

Another feature of chezmoi is the ability to put encrypted dotfiles in your public repository safely.
Chezmoi supports whole file encryption using [age](https://github.com/FiloSottile/age) or value encryption using password managers.
I don’t use password manager yet so I can’t say much about this.
But I use the whole file encryption with age and it works fine.
If you want to have a glimpse on what age is, I have a post about [how I use SOPS and age to encrypt my kubernetes secrets](https://budimanjojo.com/2021/10/23/flux-secret-management-with-sops-age/).

To actually create encrypted dotfile using age with chezmoi, you need to have this section in your chezmoi config file:

```
encryption: age
age:
  identity: <path to file with your age private key>
  recipient: <string of your age public key>
```

Then you can simply do `chezmoi add --encrypted <filepath>` to create an encrypted file in your repository.
Chezmoi will decrypt the file on your other machine before applying them using the same age private key.
You might want to have age private key inside your new machine first.

## Ending

I think this should wrap all the things I want to share about chezmoi.
It’s so much more you can do with chezmoi but I really don’t want this blogpost to be any longer.
And this is how I’m managing dotfiles with chezmoi.
Feel free to leave some comments if you have any suggestion or question.
Also check out my [dotfiles](https://github.com/budimanjojo/dotfiles) repository if you want to look at how I implement chezmoi.
