---
layout: post
title: Self hosting Renovate on GitHub
author: Tomasz Cielecki
comments: true
date: 2025-10-13 22:00:00 +0100
tags:
- ci
- GitHub
- nuget
- renovate
---

Early last year [I wrote a few blog posts about hosting Renovate Bot in Azure DevOps][renovate-post] to automatically create Pull Requests when dependencies update, [how to configure it to group updates][more-renovate-post] and [how to share configurations][configs-renovate-post].
I have recently had a look at using Renovate Bot on GitHub as well as I have some projects where Dependabot simply does not work very well out of the box and would require me to use my own GitHub Action and using their CLI tool. While this would likely work I would still have to script all the PR creation etc. which is not exactly something I would want to deal with, especially when I am really happy with how Renovate Bot works for me in Azure DevOps.

To run Renovate Bot in GitHub you have a few options.

1. You can add and run [Mend Renovate GitHub App][renovate-app]
2. You can [self host using GitHub Actions][renovate-gh-action]

The first option is great if you trust a third party scanning your repositories and creating a [dashboard of all your dependencies and more][mendio], it doesn't seem to cost anything to have the default Renovate features but mend offer more than just dependency updates.

The second option self hosting is great too and you are in control. You chose whether to run on GitHub Hosted runners or your own agents, up to you. Under the hood it [works in a similar fashion as described in my first blog post][renovate-post], with a Docker container spinning up and running Renovate. The main difference here is that the Renovate Bot team wrapped it up nicely in a GitHub Action ready to be used.

## Authentication in GitHub

Using `secrets.GITHUB_TOKEN` is really nice in GitHub Actions. However, this token has a lot of limitations and is scoped to only work in the repository you store your GitHub Action for Renovate in. Meaning that this token will not be able to create Pull Requests, Issues and comment in other repositories than its own. If you plan to run Renovate for only one repository this may work. However, if you want to run it for multiple repositories on your user or organization then you have to use some of GitHub's other options.

### Personal Access Tokens

You have probably already tried [Personal Access Tokens (PAT) for some features in GitHub][gh-pat]. If not they are simply a token tied to a specific user with a limited set of scopes it has access to and with an expiration date or optionally without any expiration.

The downside is that it is _personal_ it is tied to you. If you leave an organization, this PAT will not work anymore. If you opt to use a PAT then [make sure to read which scopes you need in their documentation][renovate-pat-scopes].

If you plan to use this PAT for repositories in an organization, which is only available for Classic tokens, remember to click the `Configure SSO` button next to your generated token and allow it for that given organization. This looks something like this:

[![gh-pat-sso][gh-pat-sso]][gh-pat-sso]

### GitHub App

Instead of using a PAT you can create your own GitHub App which you install to your organization or selectively repositories you want it to run on. When running Renovate in GitHub Actions, you will create a token for this App instead and the App will have the permissions to operate on the repositories. Also, instead of then creating Pull Requests on your behalf like with a PAT, it will show up as the Application instead. Also if you happen to leave the organization, Renovate will keep working.

I opted for this solution, even though it requires a little bit of setup. However, it is not that hard to do.

First you need to create your GitHub App. I did this on my organization going to https://github.com/organizations/MYORG/settings/apps/new

Here you just need to give it a name, such as `MYORG Renovate` and fill in any URL. I opted to link to the repository where renovate is going to live in my organization.
You can disable Webhooks.

Then the important step is to provide the GitHub App the correct permissions for Renovate to do its job. Make sure to [read which ones to check off in the Renovate docs][renovate-gh-app-permissions]. As of writing you will need:

| Permission | Scope |
|------------|-------|
| Checks | Read + Write |
| Commit statuses | Read + Write |
| Contents | Read + Write |
| Dependabot alerts | Read |
| Issues | Read + Write |
| Pull requests | Read + Write |
| Workflows | Read + Write |
| Administration | Read |
| Members | Read |

You can always add more permissions later, but they would need to be authorized per repo or org you added the App for, so better get this right to begin with.

> If you want to restore private packages from your organization using the GitHub Apps token, I recommend adding `Read` scope to `Organization Private Registries`.

Once you have created it, you will at the top of the General tab for the Application, see an `App ID`. This ID, you will need later for authentication.

There should be a prompt at the top of the page that you need to generate a Private key. Save this file as we need this later too for authentication. The contents of the file should start with something like `-----BEGIN RSA PRIVATE KEY-----` you need to include both this and the ending like when storing the secret.

I opted to store these two pieces of information in GitHub Actions Secrets, so I can refer to them as `secrets.RENOVATE_APP_ID` and `secrets.RENOVATE_PRIVATE_KEY`

Last but not least, you need to install the application to your organization and figure out whether you want to give it access to all repositories or only select. If you opt for the latter option with only select repositories, you can always add more repositories or change your mind later on the installation page of the App.

## Running Renovate in GitHub Actions

For running Renovate in my organization, I opted to create a repository in the organization called `renovate-bot` which is where I run the GitHub Action and have the default configuration stored.

For the action create a new workflow in the folder `.github/workflows` I called mine `renovate.yml`.

If you are using a PAT you only need two steps. Checkout and the renovate action. This would look something like:

{% raw %}
```yml
- name: Checkout
  uses: actions/checkout@v5

- name: Self-hosted Renovate
  uses: renovatebot/github-action@v43
  with:
    configurationFile: renovate-config.js
    token: '${{ secrets.RENOVATE_TOKEN }}'
```
{% endraw %}

However, if you are using the GitHub App authentication approach you need a little bit more config. First you need to authenticate the GitHub App

{% raw %}
```yml
- name: Get token
  id: get_token
  uses: actions/create-github-app-token@v2
  with:
    private-key: ${{ secrets.RENOVATE_PRIVATE_KEY }}
    app-id: ${{ secrets.RENOVATE_APP_ID }}
    owner: ${{ github.repository_owner }}
    repositories: |
      repo1
      repo2
```
{% endraw %}

So here you use the App ID and Private Key you generated for the GitHub App earlier.

For the repositories, you need to specify which repositories you want to authenticate the token for. You can also remove this and authenticate all the repos the GitHub App has access to. Just keep the `owner` argument to authenticate all repositories for that owner.

Then instead of the PAT you authenticate using the output from this task:

{% raw %}
```yml
- name: Self-hosted Renovate
  uses: renovatebot/github-action@v43
  with:
    configurationFile: renovate-config.js
    token: '${{ steps.get_token.outputs.token }}'
```
{% endraw %}

## Configuration

I store the `renovate-config.js` in the root of the repository and I have something like this in there:

```js
module.exports = {
  branchPrefix: 'renovate/',
  username: 'renovate-release',
  gitAuthor: 'Renovate Bot <bot@renovateapp.com>',
  onboarding: false,
  platform: 'github',
  repositories: [
    'owner/repo1',
    'owner/repo2'
  ]
};
```

This works great for most repositories that only require fetching package updates from public sources.

### Authenticating Private GitHub Packages

I spent a bunch of time figuring out how to get GitHub Packages to work for one of the projects I work on. It uses maven packages stored in GitHub Packages on a private repo.

A couple of things I struggled with which I wish I had know beforehand.

1. The `renovatebot/github-action` does not forward _all_ environment variables to the docker container that runs underneath the hood. There is a [regex that only allows some][renovate-regex] environment variables. So prefix your environment variables for Renovate with `RENOVATE_` for use in your config
2. The GitHub Token from the system `secrets.GITHUB_TOKEN` even when specifying `packages:read` permission, does not get access to other than the repo we are running in currently packages
3. Using `RENOVATE_X_GITHUB_HOST_RULES` does not work as it uses `secrets.GITHUB_TOKEN` under the hood and by design is broken, do not chase this option

With that in mind. Authenticating GitHub Packages, such as maven, npm and NuGet is fairly straight forward. You will need to add a `hostRule` to specify how to authenticate. Since we already have either a PAT or GitHub App Token (given you added Read permission for Organization Private Registries) with read permissions to packages, then we can just use that token in your rule. So in your `renovate-config.js` you can add:

```js
hostRules: [
  {
    hostType: 'maven',
    matchHost: 'maven.pkg.github.com',
    username: 'x-access-token',
    password: process.env.RENOVATE_TOKEN,
  },
],
```

This should be similar for other hostTypes too when used with GitHub Packages.

If you want to provide your own token or you are authenticating some other source. Just make sure you prefix the environment variable with `RENOVATE_`. So something like this;

{% raw %}
```yml
- name: Self-hosted Renovate
  uses: renovatebot/github-action@v43
  with:
    configurationFile: renovate-config.js
    token: '${{ steps.get_token.outputs.token }}'
  env:
    RENOVATE_MY_TOKEN: '${{ secrets.MY_TOKEN }}'
```
{% endraw %}

Then in the config you can use it as `process.env.RENOVATE_MY_TOKEN`.

## Troubleshooting

When troubleshooting Renovate, you can run it dry, so it won't open any Pull Requests by adding `dryRun: 'full'` in the config:

```js
module.exports = {
  dryRun: 'full',
  ...
```

Then you can also increase the verbosity of the log by adding the environment variable `LOG_LEVEL` with a supported log level. I.e.:

{% raw %}
```yml
- name: Self-hosted Renovate
  uses: renovatebot/github-action@v43
  with:
    configurationFile: renovate-config.js
    token: '${{ steps.get_token.outputs.token }}'
  env:
    LOG_LEVEL: debug
```
{% endraw %}

This should spit out much more information on what Renovate is doing and you can attempt to deduct what went wrong.

Eventually you should see pull requests flowing in on your repos looking something like this:

[![pr-example][pr-example]][pr-example]

> Note: you will see warnings on your pull requests like in the example above if something goes wrong when Renovate is running. This is your cue to troubleshoot these.

Otherwise the stuff in my previous renovate posts still applies and can be used on GitHub as well.

[renovate-post]: {% post_url 2024-01-11-renovate %}
[more-renovate-post]: {% post_url 2024-02-12-more-renovate %}
[configs-renovate-post]: {% post_url 2024-03-08-renovate-bot-sharable-configs %}
[mendio]: https://www.mend.io/
[renovate-gh-action]: https://github.com/renovatebot/github-action
[renovate-gh-app-permissions]: https://docs.renovatebot.com/modules/platform/github/#running-as-a-github-app
[renovate-app]: https://github.com/apps/renovate
[renovate-pat-scopes]: https://docs.renovatebot.com/modules/platform/github/#running-using-a-fine-grained-token
[gh-pat]: https://github.com/settings/tokens
[gh-pat-sso]: {{ site.url }}/assets/images/renovate-gh/gh-pat-sso.png "Screenshot of Configure SSO option on GitHub Personal Access Token page"
[renovate-regex]: https://github.com/renovatebot/github-action/blob/e8fc25c747f24032368eb5dfd40ab54491f4640c/src/input.ts#L12
[pr-example]: {{ site.url }}/assets/images/renovate-gh/pr-example.png "Screenshot of Pull Request from Renovate with a warning"