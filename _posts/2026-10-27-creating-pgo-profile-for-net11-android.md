---
layout: post
title: Creating a PGO startup profile for .NET 11 Android Apps
author: Tomasz Cielecki
comments: true
date: 2026-10-27 13:00:00 +0100
tags:
- dotnet
- maui
- android
---

In my [previous post about getting ready for .NET 11][getting-ready-for-net11] I mentioned that with CoreCLR and ReadyToRun, depending on your project you may have to generate your own startup traces. .NET MAUI ships with their own startup traces, but mainly cater for users that use MAUI UI framework in their App. So in cases you run a play Android App with Android XML Layouts, Avalonia, Uno, MvvmCross or something else you might want to create your own startup trace. You can still make your own custom trace for Apps using MAUI UI Framework for more precise profile of startup fitting your App.

Why this is important to do is, similar to .NET 10 and below Android Apps, when you were compiling for release, you would get a Ahead of Time (AOT) compiled App. However, not everything would be AOT compiled, but rather startup, based on a profile that you weren't able to easily change yourself.
.NET 11 functions in the same way where ReadyToRun is compiling primarily startup of your App Ahead of Time, but now you have the possiblity of customizing this profile. Additionally I've found with the Apps I work on the size of the App is slightly bigger than it used to be for mono runtime based Apps, but when providing my own profile it shaves off a lot of the size.

Thankfully the tooling for creating these profiles become pretty good now with .NET 11 RC2 and latest MAUI CLI tooling and the team is improving this continuously.

Lets get into creating a profile for your Android App!

## Prerequsites

Obviously this post is for .NET 11, so make sure you are on latest .NET 11 SDK. As of writing this is RC3.

You will also need to install the `maui` dotnet tool:

```bash
dotnet tool install --global Microsoft.Maui.Cli --prerelease
```

For Apps that are not running MAUI UI framework, you will also need to add a package reference to the [NuGet package `Microsoft.Maui.ProfilingHelper`](https://www.nuget.org/packages/Microsoft.Maui.ProfilingHelper) and add the line `MauiProfilingMarker.Complete();` during your startup, typically this would be in `MainActivity.OnCreate()`. This is a hint for the profiler to stop the startup profile and that your App has finished launching.

## Profiling startup

To create the profile with the `maui` cli tool you simply run:

```bash
maui profile startup --project src/Application/Application.csproj -f net11.0-android --format mibc --stopping-event-provider-name Microsoft.Maui.ProfilingHelper --stopping-event-event-name StartupComplete
```

> My Apps csproj file is in `src/Application/Application.csproj`, yours might differ and you need to adjust this path

After building your App using the `Release` configuration it should automatically launch on the device or emulator that is running in the foreground or `dotnet run` may ask you which device you want to use.

As output from the command you should see something like:

```
ℹ Project: /Users/cheesebaron/git/MyApp/src/Application/Application.csproj
ℹ Framework: net11.0-android
ℹ Configuration: Release
ℹ Device: Android_SDK_built_for_arm64 (emulator-5554)
ℹ Format: mibc
ℹ Output: /Users/cheesebaron/git/MyApp/Application_20261027_095853.mibc
ℹ Raw trace companion: /Users/cheesebaron/git/MyApp/Application_20261027_095853.nettrace
ℹ Diagnostic port: 9000
ℹ Building the app (1:46.5s)
ℹ Deploying and launching the app (5.7s)
ℹ Waiting for the configured stopping event. Press Enter to stop early.
⚠ The app did not connect to the startup profiling exit channel, so it may remain running and not flush PGO data. Ensure it references Microsoft.Maui.ProfilingHelper and loads that assembly during startup.
ℹ Converting the raw trace to MIBC...
✓ Startup trace saved to /Users/cheesebaron/git/MyApp/Application_20261027_095853.mibc (raw .nettrace companion: /Users/cheesebaron/git/MyApp/Application_20261027_095853.nettrace)
```

> If it doesn't save a trace, it didn't pick up the `Microsoft.Maui.ProfilingHelper` marker and you may have to figure out what is going on there.

The `.mibc` file is the important one here, and this is the one you will need to add a reference to in your Application csproj file like so:

```xml
<ItemGroup>
  <_ReadyToRunPgoFiles Include="..\..\pgo\android\*.mibc" />
</ItemGroup>
```

The path is relative to where your application csproj lives and you need to point it correctly to it.

Once you've done this you are ready to release your App!

## Visualizing the startup trace

If you want to visualize the traces, you can use the `dotnet trace` tool to convert. Get it with

```bash
dotnet tool install --global dotnet-trace
```

Then you can convert the `.nettrace` file that is also produce to a speedscope with:

```bash
dotnet trace convert Application_20261027_095853.nettrace --formtat Speedscope
```

You can open the newly generated `.speedscope.json` on [the Speedscope webpage](https://www.speedscope.app/) and you will see something like in this screenshot, showing a flame graph, which in my case shows Microsoft Extensions Dependency Injection speding some time resolving dependencies and eventually running some of my `MainActivity` calls at the end of the graph.

[![Screenshot speedscope webpage showing flame graph of the App startup][speedscope]][speedscope]

[speedscope]: {{ site.url }}/assets/images/pgo-profile/speedscope.png "Screenshot speedscope webpage showing flame graph of the App startup"
[getting-ready-for-net11]: {% post_url 2026-10-26-getting-ready-for-net11 %}