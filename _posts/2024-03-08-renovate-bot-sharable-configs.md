---
layout: post
title: Renovate Bot Sharable Configurations
author: Tomasz Cielecki
comments: true
date: 2024-03-08 22:00:00 +0200
tags:
- ci
- azure devops
- nuget
---

If you haven't already noticed by the amount of blog posts about Renovate Bot, I am really loving it and its feature set.

One very cool feature that was pointed out to me was the ability to have defaults or configurations to extend from, shared from a repository. I put mine in the repository Renovate Bot lives in and share it from there.

So, if you find yourself applying the same configuration in multiple repositories, this is maybe something you want to look into.

## Defining a default config

Where your Renovate Bot lives, I created a `defaults.json`, but you can actually call it almost anything you want, you will need remember the name though for when extending it in your config for your repos you are scanning. With the file `defaults.json` in place. In this file I put something like this as these are things I keep applying most places:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],
  "prHourlyLimit": 0,
  "prConcurrentLimit": 0,
  "automerge": true,
  "azureWorkItemId": 123456,
  "labels": [
    "dependencies"
  ]
}
```

## Using default configs in your configs for your repositories

To use the above `defaults.json` it is as easy to remove the configuration entries that you want to use from the defaults and adding a line such as this to your `renovate.json` config in the scanned repository.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>MyProjectName/RenovateBot:defaults"],
  ...
```

So here I use `local>` which means self hosted Git. If you are on GitHub or GitLab or some other hosted Git please refer to the [Preset Hosting documentation](https://docs.renovatebot.com/config-presets/#preset-hosting). For Azure DevOps Repositories, `local>` works.

Otherwise, I just specify the Project name in Azure DevOps and the Repository the configuration I want to extend lives in. That is more or less it.

Next time your Renovate Bot runs, it will pull those config items.

There is a lot more you can do along with some recommended presets by the Renovate Bot team which you can apply. Read more about it in their [documentation about Sharable Config Presets](https://docs.renovatebot.com/config-presets).
