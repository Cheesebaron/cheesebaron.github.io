---
layout: post
title: Consuming private Swift Packages in GitHub Actions
author: Tomasz Cielecki
comments: true
date: 2025-02-19 8:30:00 +0100
tags:
- Azure DevOps
- GitHub
---

I had a case for a native App we are working on where we already have some Swift Packages in Azure DevOps Repos, which
we would like to consume in a project that lives in GitHub.

Locally on your machine this setup is pretty easy to work with if you are using something like [Git Credential Manager][gcm].
You just install the manager, use https urls and it will pop open a Web Browser to ask for your credentials when needed.
This interactive way of authenticating is not really possible when you run in CI.

## Stuff that didn't work

I tried using git credential store and added something like this in the beginning of my workflow:

```yml
- name: Add Azure DevOps repo authentication for SwiftPM
  run: |
    git config --global credential.helper store
    git config --global --add http.https://$ORG@dev.azure.com/$ORG/$PROJECT/_git/.extraHeader "AUTHORIZATION: Basic $BASE64PAT"
  env:
    ORG: orgname
    PROJECT: projectname
    BASE64PAT: ${{ secrets.AZURE_DEVOPS_REPO_PAT_BASE64 }}
```

This didn't work at all, even though multiple source online says it was supposed to.

I also thought of using SSH keys, but I don't want to do that since Azure DevOps Repos do not support LFS over SSH, so
that would open up another can of worms for me.

## `.netrc` to the rescue

[`.netrc`][netrc] is know from the *nix world for allowing you to store credentials for automatically log into services like ftp,
http etc. So a help to avoid having to enter your credentials every time you want to connect to these services.

In GitHub Actions, there is a convenient action to create such file for you:

```yml
- uses: extractions/netrc@v1
  with:
    machine: dev.azure.com
    username: orgname
    password: ${{ secrets.AZURE_DEVOPS_REPO_PAT }}
```

Adding the `.netrc` now allows SwiftPM to resolve the package and CI is now happy

While this works for Swift Packages, it will likely work for other things for you as well.

[gcm]: https://github.com/git-ecosystem/git-credential-manager Git Credential Manager GitHub repository
[netrc]: https://everything.curl.dev/usingcurl/netrc.html More information about netrc
