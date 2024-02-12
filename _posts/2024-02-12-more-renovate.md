---
layout: post
title: Cool Renovate Bot Features
author: Tomasz Cielecki
comments: true
date: 2024-02-12 15:00:00 +0200
tags:
- ci
- azure devops
- nuget
---

Last month [I wrote about how cool Renovate Bot is][post], updating the dependencies in your repositories. It works anywhere, I got it to work in Azure DevOps Pipelines running every night at 3 AM, while everyone is sleeping and no one else is using the build agents for important stuff.

I initially did a couple of things, that I changed later on, after getting a bit more familiar with how Renovate Bot works. I will go through some of the discoveries I did and how I configure it in the end, in this blog post. Hopefully you find this useful 😀

## Configuration and pipeline can live in its own repo

Initially I had the renovate bot configuration and pipeline living in the same repository of the code I wanted it to run against. This is entirely not necessary and it can live in its own repository and have a base configuration, for things such as authentication in this repository.

So now I have a repository called `RenovateBot` with two files in it:

- `azure-pipelines.yml` the pipeline for running the bot (see [the previous post][post] on how I set that up)
- `config.js` the configuration for the bot

When running, Renovate already knows how to check out files in the repositories you tell it to scan, in the config file. So you don't need to run it in the code repositories, you want to scan for updates.

In the `config.js` file I now simply have something like:

```js
module.exports = {
    hostRules: [
        {
            hostType: 'nuget',
            matchHost: 'https://pkgs.dev.azure.com/myorg/',
            username: 'user',
            password: process.env.NUGET_TOKEN
        },
    ],
    repositories: [
        'myorg/repo1',
        'myorg/repo2',
        'myorg/repo3'
    ]
};
```

It will scan all those repositories defined in the `repositories` collection.

Neat!

## You can have repository specific configs

For each repository you define, apart from the basic configuration you provide for renovate, you can add additional configuration. I use this to add tags and group Pull Requests made by Renovate for dependencies that group together. For instance for Unit Test dependencies.

So in each repository you can add a `renovate.json` file with additional configuration. This is the same file that Renovate creates initially on a repository on the first Pull Request it makes.

Here is an example of what a configuration for one of my repositories looks like:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "azureWorkItemId": 123456,
  "prHourlyLimit": 0,
  "labels": ["dependencies"],
  "automerge": true,
  "packageRules": [
    {
      "matchPackagePatterns": [
        "Cake.*",
        "dotnet-sonarscanner",
        "dotnet-reportgenerator-globaltool"
      ],
      "groupName": "CI",
      "addLabels": [
        "CI"
      ]
    },
    {
      "matchPackagePatterns": [
        ".*xunit.*",
        "Moq.*",
        "AutoFixture.*",
        "liquidtestreports.*",
        "Microsoft.NET.Test.SDK",
        "Microsoft.Reactive.Testing",
        "MvvmCross.Tests",
        "Xamarin.UITest",
        "coverlet.*",
        "MSTest.*",
        "System.IO.Abstractions.*"
      ],
      "groupName": "Unit Test",
      "addLabels": [
        "Unit Test"
      ]
    }
}
```

Let's go through some of the options here.

- `azureWorkItemId` will add the specific work item Id to every Pull Request it creates. This is especially useful if you have a policy set on your Pull Request to always link a work item
- `prHourlyLimit` I've set this one to 0, such that Renovate Bot can create as many Pull Requests it wants on a repository. Otherwise, I think the default is 2. So if you wonder why it didn't update all dependencies, this could by why
- `labels` This option lets you set default labels on pull requests, so for each of my Pull Requests made by Renovate it will have the `dependencies` label on it
- `automerge` This option will set Auto Complete in Azure DevOps on a Pull Request using the default merge strategy, such that you can have Pull Requests automatically merge when all checks are completed
- `packageRules` Is super powerful. Here you can limit which packages you want to be grouped together, in the case above I have two groups. `Unit Test` and `CI`, which will look for specific regex patterns of package names to include in the groups. I also add additional labels for these two groups using `addLabels` and assign `groupName` such that when Renovate creates a Pull Request for a group, the title will be `Update <group name>`. There are many more options you can set on `packageRules`, you should refer to [the docs if you want more info][docs].

### You can scan many types of project types

So far I have scanned .NET projects and Kotlin projects with Renovate Bot and it handles these very well without any issues. I simply add additional repositories in the `config.js` file and on next run or when I run the pipeline manually it adds a `renovate.json` file to the repository and it is good to go.

### Some Azure DevOps annoyances

When using `System.AccessToken` as your Renovate Token, the Pull Requests are opened by the user `Project Collection Build Service (myorg)`. This user is built into Azure DevOps and does not have any e-mail assigned to it and you cannot change it either. If you have "Commit author email validation" enabled on a repo, you will need to add both the renovate bot email (or the one you've defined in your config) along with the Project Collection user like so: `renovate@whitesourcesoftware.com; Project Collection Build Service (myorg)` to the allowed commit author email patterns. Otherwise auto completion on Pull Requests will not work as it will violate one of the repository policies.

[post]: {% post_url 2024-01-11-renovate %}
[docs]: https://docs.renovatebot.com/configuration-options/#packagerules "Renovate Docs about packageRules"
