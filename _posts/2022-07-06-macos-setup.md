---
layout: post
title: Easy Setup of Dev Tools on macOS with ZSH dotfiles
author: Tomasz Cielecki
comments: true
date: 2022-07-06 16:30:00 +0200
tags:
- macOS
---

I recently got a new machine at work. Before I got it I spent a bit of time preparing and figuring out which essential tools I need to do my daily work. But, also to minimize the amount of time I would have to spend installing everything.

As you the reader might know I develop mobile Apps with Xamarin and .NET. So this blog post will be geared towards that. However, you would be able to install anything using the same approach.

My colleague pointed out that there is this super cool feature of ZSH called dotfiles. ZSH ships with macOS ans is the shell that you see when you open a terminal. It has a set of very powerful interactive tools but is also a script interpreter. Similar to other shells you might know such as bash, fish and csh.

The configuration for ZSH happens with a file called `.zshrc`, which is a list of settings, aliases for commands, styles and more.

Dotfiles in ZSH is a way to organize the settings you would normally put it `.zshrc`, but not only that, you can organize other configuration files for other tools, and also organize scripts.

There are multiple attempts to build on top of these dotfiles for easily managing the tools you need for your daily work. The version I went for is [Oh Your dotfiles][ohyourdotfiles]. With this tool you can make a logical folder structure with scripts and dependencies you want to have installed.

Let's dig into it!

Start off by cloning [Oh Your dotfiles][ohyourdotfiles]:

```sh
git clone https://github.com/DanielThomas/oh-your-dotfiles ~/.oh-your-dotfiles
```

Then you run

```sh
ZDOTDIR=~/.oh-your-dotfiles zsh
```

Followed by:

```sh
dotfiles_install
```

Now you have oh your dotfiles installed. Time to create your own repository to customize your installation. For my own use I've created a [dotfiles][dotfiles] repository, with the tools I need.
I've called it dotfiles on purpose as Oh Your Dotfiles automatically looks for paths containing `dotfiles` in the name, call it anything you want, just make sure you clone it into a folder starting with `.` and containing `dotfiles` in your user directory.

Now with the stuff cloned, you can start add content.

For instance. I have a folder called `dev` with a script called `install.homebrew` with the following contents:

```sh
ca-certificates
git
gh
scrcpy
python@3.9
mitmproxy
gawk
imagemagick
curl
wget
tree
```

This will install all those tools from homebrew.

Similarly if you want to install stuff from homebrew casks you can create a `install.homebrew-cask` file and add stuff there. For instance I have this in one of my folders:

```sh
firefox
amethyst
anydesk
little-snitch
spotify
gimp
signal
macfuse
keka
hot
```

When you've added some files and stuff to download. Simply run `dotfiles_update` and it will install everything.

Remember to synchronize your dotfiles somewhere. Then next time you get a new machine you can simply do:

```sh
git clone https://github.com/DanielThomas/oh-your-dotfiles ~/.oh-your-dotfiles
git clone https://github.com/Cheesebaron/dotfiles ~/.dotfiles
ZDOTDIR=~/.oh-your-dotfiles zsh
dotfiles_install
```

Then you can wait and everything is installed. Super convenient.

[ohyourdotfiles]: https://github.com/DanielThomas/oh-your-dotfiles
[dotfiles]: https://github.com/Cheesebaron/dotfiles