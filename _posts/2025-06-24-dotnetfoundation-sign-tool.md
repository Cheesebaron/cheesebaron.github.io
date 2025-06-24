---
layout: post
title: Migrating Signing of NuGet packages to new sign tool
author: Tomasz Cielecki
comments: true
date: 2025-05-24 10:30:00 +0100
tags:
- NuGet
- dotnet
---

I maintain [MvvmCross][mvx] which is a part of the [.NET Foundation][dnf] and one of the services member projects get is signing of software using certificates issued by the .NET Foundation. This way the project does not have to manage their own signing certificates and not have to spend money on these.

So when I publish NuGet packages for MvvmCross when merging to develop or when releasing a stable release, these NuGet packages are signed. This way consumers of the packages can validate the authenticity of the package and know that it has not been tampered with.

Historically the way signing worked was using the [SignClient][signclient] .NET tool, which is now depcrecated as well. The .NET Foundation has also moved over to use Azure Key Vault for their certificates, so a new tooling is required for member projects to sign their packages.

With help from the .NET foundation team, I have managed to get MvvmCross packages signed again after the deprecation of the other tool. Which was suprisingly straight forward. MvvmCross uses GitHub Actions Windows runners and signing is now done by:

1. Download the new [`sign`][sign] tool
2. Sign in to Azure CLI
3. Sign packages which are in the `${{ github.workspace}}/output` folder

```yaml
- name: Install sign tool
  run: dotnet tool install --tool-path . sign --version 0.9.1-beta.25278.1

- name: 'Az CLI login'
  uses: azure/login@a457da9ea143d694b1b9c7c869ebb04ebe844ef5 #v2.3.0
  with:
    client-id: ${{ secrets.SIGN_AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.SIGN_AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.SIGN_AZURE_SUBSCRIPTION_ID }}

- name: Sign NuGet packages
  shell: pwsh
  run: >
    ./sign code azure-key-vault
    **/*.nupkg
    --base-directory "${{ github.workspace }}/output"
    --publisher-name "MvvmCross"
    --description "MvvmCross is a cross platform MVVM framework"
    --description-url "https://mvvmcross.com"
    --azure-key-vault-url "${{ secrets.SIGN_AZURE_VAULT_URL }}"
    --azure-key-vault-certificate "${{ secrets.SIGN_AZURE_KEY_VAULT_CERTIFICATE_ID }}"
```

The secrets are provided to you by the .NET Foundation.

You can double check that the package has been signed either using the NuGet package explorer and upload your signed package there. Which on the left side will show a "Digital Signatures" section showing something like:

[![Screenshot of NuGet Info showing Digital Signatures pane][nugetinfoss]][nugetinfoss]

[mvx]: https://github.com/mvvmcross/mvvmcross "MvvmCross is an opinionated cross-platform MVVM framework for .NET Apps"
[dnf]: https://dotnetfoundation.org
[signclient]: https://www.nuget.org/packages/SignClient
[sign]: https://www.nuget.org/packages/sign
[nugetinfo]: https://nuget.info/
[nugetinfoss]: {{ site.url }}/assets/images/sign/nugetinfo.png "Screenshot of NuGet Info showing Digital Signatures pane"