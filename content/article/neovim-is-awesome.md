---
title: "Neovim Is Awesome"
date: 2022-08-27T12:20:51+07:00

categories:
  - Terminal stuffs
  - Desktop stuffs
  - Linux
tags:
  - neovim
  - lua
  - vim
  - editor
  - terminal
toc: false
author: budimanjojo
slug: neovim-is-awesome
---
![neovim-is-awesome](/images/neovim-is-awesome_1.png)

I was a long time [Vim](https://www.vim.org/) user, long before [Neovim](https://neovim.io/) was born.
I was sold to Neovim when [CoC](https://github.com/neoclide/coc.nvim) came out and there were some features that don't work with Vim.
At that time, Neovim was marketed as Vim but with more sane default experience.
As time goes by, now Neovim is more than just Vim with better defaults.
In this post, I will share my two cents on why Neovim is so much better than Vim, at least for me.
<!--more-->

## Background

Neovim was born as a derivative/clone of Vim with focus on enabling new contributors.
There is no [benevolent dictator](https://en.wikipedia.org/wiki/Benevolent_dictator_for_life) in Neovim like Vim has [Bram Moolenaar](https://en.wikipedia.org/wiki/Bram_Moolenaar).
And that's the biggest difference between them.
That's it?
Yes, until that small difference made so much more differences.

As I mentioned above, I started using Neovim just because I wanted to try out some floating window in CoC.
I didn't notice any difference between using Neovim and Vim at that time.
I did removed a big chunk of my `vimrc` because there are a lot of settings I have are the defaults in Nvim.
And I was happy with the move.

## v0.5.0 Happened

Although I was happy with nvim + CoC combination, Neovim was never special for me.
I didn't think much about it, I didn't even know what version of Neovim I was at.
It was just Vim with `:syntax on` and `:filetype on` by default to me.

That all changed when they released [v0.5.0](https://github.com/neovim/neovim/releases/tag/v0.5.0).
The hype behind the [builtin LSP client](https://neovim.io/doc/user/lsp.html) was so big I heard people talking about it everywhere.
And that led me to dig more information about it and I was surprised by how great Neovim is with that release.
Native LSP client plus beta Treesitter plus Lua user config together in a package, who can resist that?

I decided to try out the builtin LSP, this means I need to get rid of CoC which was awesome at that time.
While doing this, I automatically learned a little bit of [Lua](https://www.lua.org/) and loved it.
Then I found the awesome [nvim-lua-guide](https://github.com/nanotee/nvim-lua-guide) and [awesome-neovim](https://github.com/rockerBOO/awesome-neovim).
At last, **I ditched CoC for native LSP and moved everything to Lua in December 2021** per this [commit](https://github.com/budimanjojo/dotfiles/commit/ab57781d13fb7ec467b170072e39363521f8d9fe).

## It Keeps Getting Better

You might be hearing a lot of "When you thought it's over" memes, but this was exactly what Neovim is doing.
Neovim never cease to surprise us with awesome new stuffs even after the legendary `v0.5.0`.
The Lua api is growing fast, that by [v0.7.0](https://github.com/neovim/neovim/releases/tag/v0.7.0) I have managed to have a 100% Lua `nvimrc`.

This is all possible because of the small difference between Neovim and Vim.
Having more than one person to make the call increase the probability of making a good decision.
All the great defaults by Neovim is one example of that, the community knows what the community wants.

## Nothing Is Perfect

I have been talking about all the good things up until here.
Now let's talk about the bad.

Because Neovim is moving so fast, plugins need to keep up with the moves.
And because I have so many plugins installed, I see a lot of plugins not working for certain version.
For example, I used to have the [Neovim Unstable PPA](https://launchpad.net/~neovim-ppa/+archive/ubuntu/unstable) in my Ubuntu machine.
I've seen plugins sometimes only support the stable version while some only support the unstable version.

Another thing I don't like about Neovim is they tend to have a lot ways to do one thing.
For example, you have `vim.api.nvim_set_keymap` and `vim.keymap.set` to set keymaps in Lua.
Both do the same thing, but one of them can call Lua function while the other can't.
This can get really confusing to the users.

## Closing

The conclusion is, I love Neovim.
I see a lot of VSCode users switching to Neovim because of the native LSP.
If you're one of them, here's my [dotfiles repo](https://github.com/budimanjojo/dotfiles) for you to peek at.

Thanks for reading, please leave some comments or reactions below.
I will be so happy if you do.
