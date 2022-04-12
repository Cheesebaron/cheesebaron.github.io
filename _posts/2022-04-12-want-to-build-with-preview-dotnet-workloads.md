---
layout: post
title: Using Android Tiramisu Preview Workloads in Azure Pipelines
author: Tomasz Cielecki
comments: true
date: 2022-04-12 11:00:00 +0200
tags:
- Xamarin
- dotnet
- Azure DevOps
---

Had the pleasure of hitting a bug in the `net6.0-android` TFM when targeting Android 31. This bug had been fixed in a preview version of the `net6.0-android` workload, targeting Android 32 or higher.

So question is, how would you set up your CI or even your local environment to run this?

In my specific case I wanted:

- Android platform for `android-Tiramisu`
- .NET workload for android, containing the fix, `android-33`

The steps to install these are fairly simple. Use Android SDK manager to install the platform. Then use dotnet to install the specific workload.

```bash
sdkmanager --install "platforms;android-Tiramisu"

dotnet workload install android-33
```

This enables me to use `net6.0-android33.0` as TFM.

`sdkmanager` is located in your Android SDK folder to invoke it, you might need to add it to your path or cd into the folder `<android-sdk>/cmdline-tools/latest/bin` and run `./sdkmanager`.

Setting up steps in Azure Pipelines is super easy too:

```yaml
- script: |
    ${ANDROID_SDK_ROOT}/cmdline-tools/latest/bin/sdkmanager --install "platforms;android-Tiramisu"
  displayName: Install Android SDK for Tiramisu

- task: UseDotNet@2
  displayName: 'Use .NET Core SDK 6.0.x'
  inputs:
    version: 6.0.x

- script: |
    dotnet workload install android
    dotnet workload install android-33
  displayName: Install Android Workloads
```

Now you should be able to build `.net6.0-android33.0` TFMs!
