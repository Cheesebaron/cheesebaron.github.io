---
layout: post
title: Restore NuGets From Private Feed With Cake
author: Tomasz Cielecki
comments: true
date: 2017-11-17 17:25:00 +0200
tags:
- Cake
- NuGet
---

I have just been wrestling with Cake, NuGet and the private feed I have for some of the stuff part of a build I am working on. When restoring stuff from private feeds with NuGet there are a couple of things you need to know about and put into consideration.

So there are basically two places where NuGet is used in Cake.

1. When running the build script it restores NuGet packages for Tools and AddIns
2. When you restore NuGet packages for a Solution or Project

For NuGet.exe to figure out how to contact a private feed you need to provide a configuration. It would usually look something in the lines of:

```xml
<configuration>
    <packageSources>
        <add key="PrivateRepo" value="https://some.url" />
    </packageSources>
</configuration>
```

For the private package sources you would additionally have a section with credentials or API keys like:

```xml
<packageSourceCredentials>
    <PrivateRepo>
        <add key="Username" value="cheesebaron" />
        <add key="ClearTextPassword" value="super_secret_passw0rd" />
    </PrivateRepo>
</packageSourceCredentials>
```

You can read more about the schema and each section in the [official NuGet documentation for the NuGet.config](https://docs.microsoft.com/en-us/nuget/schema/nuget-config-file).

## 1. Restoring NuGet packages for Tools and AddIns in Cake
So a couple of things to think about when restoring NuGets for tools and AddIns in Cake, is how you specify them.

Usually you simply add something like the following in the top of your cake script:

```csharp
#tool nuget:?package=MyPackage
```

This will tell NuGet to look in _all_ configured sources for `MyPackage` latest stable version.

You can control which source to look explicitly in by specifying the source before the `?` like so:

```csharp
#tool nuget:https://some.url?package=MyPackage
```

When you explicitly set the source, NuGet.exe will _only_ look for packages in this source. Meaning, if your package has other dependencies, lets say it depends on `System.IO.Compression` from NuGet and your private repository does not contain this package or does not proxy it. NuGet will not be able to restore this dependency and fail!

As for credentials and sources specified in the nuget.config, Cake will automatically pick this up, if the nuget.config is next to your bootstrap `build.ps1` file. Previously I had mine in `.nuget/nuget.config` and nothing seemed to work, moving it helped.

## 2. Restoring NuGet packages for solutions and projects
After moving the `nuget.config` to where the build script executes from, it seems like this is applied too for restoring packages when using the `NuGetRestore` tool Cake provides out of the box.

This can always be overridden by another configuration if desired, by providing `NuGetRestoreSettings`:

```csharp
var nugetRestoreSettings = new NuGetRestoreSettings {
    ConfigFile = new FilePath("/path/to/nuget.config")
};

NuGetRestore(sln, nugetRestoreSettings);
```

## Cake In Process NuGet Client
Now as of Cake 0.22.0 there is something called In Process NuGet Client, which you can enable by adding:

```powershell
$ENV:CAKE_NUGET_USEINPROCESSCLIENT='true'
```

To your `build.ps1` or by using the `--nuget_useinprocessclient=true` argument. Read more about configuring it in the [Cake docs for Default Configuration Values](https://cakebuild.net/docs/fundamentals/default-configuration-values).

All the above seems to also apply to the In Process Client.