---
layout: post
title: Getting Your Apps Ready For .NET 11
author: Tomasz Cielecki
comments: true
date: 2026-10-26 15:00:00 +0100
tags:
- dotnet
- maui
- mvvmcross
- android
- ios
---

.NET 11 is right around the corner. As of writing this post .NET 11 RC3 is out and ready to use and RC1 and up has the Go-live stamp from Microsoft, which means you should be OK to use this in production and not expect major changes before general availability. So this should be a sign for you to start looking into getting your Apps upgraded.

For mobile workloads in .NET there are some very big changes being released. The biggest change is that the runtime is no longer mono and is now CoreCLR! So there are some changes on how Apps are built and running on device. Which you would likely need to test and see if this affects your App.

Since .NET 11 iOS SDK is targeting iOS 27 SDK you might have to migrate to `UISceneDelegate` instead of many of the deprecated methods in `UIApplicationDelegate`, as these won't longer work with Xcode 27.

But let me go through some of the things you need to be aware off.

# Android

## CoreCLR being the default on Android

With .NET 11 the runtime is being switched away from Mono to CoreCLR, which brings quite a few changes. The switch to CoreCLR means that the runtime will be the same across all the platforms you will target, whether it is Linux, Windows or Android, which is very cool.

> If you want to opt out CoreCLR you can do this with: `<UseMonoRuntime>true</UseMonoRuntime>`, but it will likely get removed in the future

For Android, CoreCLR release builds no longer include diagnostic libraries, which helps reduce APK size. The Android workload will also report an error when Mono-specific AOT settings are used in a CoreCLR project. Make sure to review any runtime-specific MSBuild properties before upgrading rather than carrying them forward blindly.

For instance, if you have `UseInterpreter` or `MtouchInterpreter` set to true, these will not work with CoreCLR as it uses a different interpreter. The flags will still affect NativeAOT builds.

The newer trimmable typemap is becoming the standard path, particularly for NativeAOT. This reduces reflection and supports smaller, more trim-friendly apps, but it exposes unsupported Java interop patterns as build diagnostics. Make sure to test apps that use custom `IJavaObject` types, JNI callbacks, `[Export]` or `[ExportField]`, or third-party Android bindings with trimming and NativeAOT enabled.

CoreCLR Release builds now use partial ReadyToRun compilation by default, improving startup at a small package-size cost. You can opt into full ReadyToRun with `<MauiEnableFullReadyToRun>true</MauiEnableFullReadyToRun>`, trading more package size for additional startup performance. RC2 adds an important condition: ReadyToRun is disabled when `PublishTrimmed=false`, with warning `XA0119`, so enable trimming if you expect ReadyToRun to be used.

> ReadyToRun (R2R) is the new terminology of Ahead of Time (AOT) compiling a CoreCLR App which has different modes depending on which platform your are targeting and properties set in your project. I.e. for Android partial ReadyToRun is the default where parts of the App is AOT compiled, mainly startup code, where rest is JIT compiled for App size and startup times.

For Apps that do not use MAUI UI framework you might have to create your own `.mibc` profile instead of using the default one that ships with the workloads. I will cover this in a future blog post. But have a look at the [`dotnet-pgo` tool and docs](https://github.com/dotnet/runtime/blob/main/docs/design/features/dotnet-pgo.md) and there are [some instructions in this maui-labs PR to automate this with AI](https://github.com/dotnet/maui-labs/pull/111).

If you are consuming libraries that are using the Xamarin templates for libraries. So not targeting `net6.0-android` or above, then you will get warnings when building your project.

## Removal of support for Android x86

With .NET 11 the RID `android-x86` is no longer supported. The supported RIDs are:

- `android-arm` (armeabi-v7a)
- `android-arm64`

## Minimum Android API level

With .NET 11 Preview 5, there was a push to consolidate the minimum supported API level which has been set to 24. Previously Mono would support API 21 as the minimum, this is not the case anymore.

Existing projects need to update `SupportedOSPlatformVersion` to `24.0`. This also means Android 6.0 and lower are no longer supported, even when opting out of CoreCLR and using Mono.

## Android API 37 is the default target

.NET 11 targets Android API 37 by default. API 37 was introduced as a preview in .NET 11 Preview 3 and became stable in Preview 5, so there is no longer a need to opt into preview features to use it. RC2 also adds API 37.2 bindings, available by targeting `net11.0-android37.2`.

## R8 now optimizes and obfuscates private Java members

When R8 is enabled, .NET 11 defaults `AndroidR8ObfuscationMode` to `private-members`. R8 optimization is also enabled, so private and package-private Java members can be renamed; Java class names and public and protected member names remain preserved for managed-to-Java interoperability.

This should reduce package size, but test code that looks up Java members by name, reflection-heavy libraries, and custom ProGuard/R8 rules. Set `AndroidR8ObfuscationMode` to `disabled` to keep the previous shrinking-only behavior while investigating a compatibility issue.

## HTTP exceptions and cancellation now match .NET

`AndroidMessageHandler` now behaves more like `SocketsHttpHandler`: transport and protocol failures throw `HttpRequestException`, while the legacy `WebException` remains available as the inner exception. Cancellation during uploads and response reads now throws `OperationCanceledException`.

Review Android-specific networking code that catches `WebException`, checks its status directly, or has separate cancellation handling. The improved behavior is more consistent with the rest of .NET, but existing exception filters may need updating. This release also fixes redirect handling for `HEAD` requests and a response-disposal crash that could affect cancelled gRPC server streams.

`AndroidHttpClientHandlerType` property is no longer supported for Android projects and the newer `UseNativeHttpHandler` which was introduced with .NET 8.

## Faster development, builds, and packages

`dotnet run` can discover and boot an Android emulator, while `dotnet watch` can deploy to Android devices and emulators and apply Hot Reload changes. `FastDeploy2` is now the default fast-deployment path, reducing the work needed for incremental installs, and CoreCLR Fast Deployment stack traces now include file and line information.

RC1 and RC2 include substantial incremental-build improvements, including faster type-map generation and avoiding unnecessary universal APK generation on unchanged Release builds. RC2 also uses stronger Zstandard compression for the assembly store, reducing the measured sample APK size by about 4.7%. These are automatic improvements, but large projects should validate their CI build timings and APK sizes after upgrading.

Android projects support JDK 21 and JDK 25, with JDK 21 as the minimum.

## Test manifest and packaging assumptions

Android builds now use response files when invoking R8, avoiding Windows command-line length failures in large projects. Legacy Xamarin.Android libraries that embed `__AndroidEnvironment__` resources now produce warning `XA0149`, and invalid global R8 options in third-party `.aar` consumer rules are skipped rather than breaking the build.

There is also a subtle manifest behavior change: if a partial `<uses-sdk>` element does not specify `android:targetSdkVersion`, the build now writes the value from `$(TargetSdkVersion)`. Test Android behaviors gated by target SDK, or explicitly set `android:targetSdkVersion` if you need to control it.

# iOS

## Xcode 27 and the UIKit scene lifecycle

.NET 11 RC2 requires Xcode 27.0 and includes updated bindings for the corresponding Apple SDKs. The most important migration consequence is that iOS and Mac Catalyst apps built with these SDKs must use the UIKit scene lifecycle.

New MAUI templates include a `UIApplicationSceneManifest` and a registered `MauiUISceneDelegate`. Existing apps need to add the manifest configuration and move custom lifecycle and launch-option handling from `UIApplicationDelegate` to scene callbacks. This is especially important for deep links, push notifications, user activities, and quick actions. A quick-action handler must call its completion callback, returning `false` when it does not handle the action.

For MvvmCross Apps you can use [`MvxSceneDelegate` and `MvxSceneApplicationDelegate` to set this up which is available as of version 9.4.0](https://www.mvvmcross.com/documentation/platform/ios/ios-scenedelegate#3-add-a-scenedelegate-class-using-mvxscenedelegate).

The iOS minimum version remains 15.0. Mac Catalyst is a separate concern: new templates require 17.0 because that is the minimum accepted by the Mac Catalyst 27 SDK.

## CoreCLR is now the default runtime

CoreCLR is the default runtime for .NET 11 iOS, Mac Catalyst, macOS, and tvOS builds. This unifies the runtime with the rest of .NET and is accompanied by build, startup, package-size, diagnostics, and Hot Reload work.

The change is substantial enough that every app should be tested on a physical device, not only a simulator. At the start of the .NET 11 previews, mobile debugging was not available with CoreCLR; if you need to remain on Mono while you investigate a runtime or debugger issue, set `UseMonoRuntime` to `true` for the iOS target. Treat that as a temporary compatibility option rather than a long-term solution.

As described in the Android section, if you have `UseInterpreter` or `MtouchInterpreter` set to true, these will not work with CoreCLR as it uses a different interpreter. The flags will still affect NativeAOT builds.

## Trimming, registrars, and dynamic code need a review

NativeAOT and CoreCLR .NET 11 apps now default to the trimmable-static registrar and assembly preparation. This improves the Apple build pipeline and enables better trimming, but it can expose assumptions made by custom bindings and reflection-heavy libraries. Test trimmed Release builds early, especially if you own Objective-C bindings or depend on libraries that generate code dynamically.

CoreCLR apps on iOS, tvOS, and Mac Catalyst now inherit `DynamicCodeSupport=true`. This allows libraries such as Entity Framework Core to use runtime model creation instead of selecting their NativeAOT path, but it also keeps `System.Reflection.Emit` in trimmed output and can increase app size. Apps that do not need dynamic code can set `DynamicCodeSupport` to `false`; NativeAOT remains `false` by default.

## Packaging and diagnostics

IPA builds now include Apple's `*.symbols` files by default, allowing App Store Connect and Xcode Organizer to symbolicate crash reports automatically. Set `IpaIncludeSymbols=false` only when you have a reason to omit them. `EnableCrashReport` also enables the .NET in-process crash-report writer for CoreCLR iOS, tvOS, and Mac Catalyst apps.

Apple builds strip `Headers`, `PrivateHeaders`, and `Modules` directories from bundled frameworks before code signing. This is normally desirable, but apps that need those directories at runtime must set `StripFrameworkHeaders=false` and verify their signing and packaging output.

## Networking and app extensions

`NSUrlSessionHandler` now supports HTTP Digest authentication, follows redirects more consistently, and supports custom proxies through `Proxy` or `UseProxy=false`. A handler selects its proxy from the first request destination, so use separate handlers when different destinations need different proxies. Proxy credentials apply to HTTPS destinations through CONNECT tunnels, not plain HTTP.

Referenced extension projects are now built automatically. Projects with app extensions should remove any redundant manual build step and verify that archives contain the expected extensions.

## Building .NET 11 iOS projects in Azure DevOps Pipelines

If you want to build .NET 11 iOS projects in Azure DevOps pipelines, you will have to switch to the pay-as-you-go GitHub Actions Runners, which are now available in Azure DevOps pipelines. This might have to be enabled for your organization. Note that the pricing of these runners are much different from the parallel runners you would normally pay for in Azure DevOps. So for these runners you pay per minute, instead of parallel instances.

To run on a runner with Xcode 27 you would need to switch to the pool like so:

```yaml
pool:
  name: 'GitHub-hosted Agents'
  vmImage: 'xcode-27'
```

You have to specify the pool name as above, unless you have your own runners named `xcode-27`.

# Testing

.NET 11 also supports running Android and iOS specific tests with `dotnet test` on a device or emulator/simulator, with `androidtest` and `iostest` and more project templates available. The templates will create a new MSTest project targeting the platform. [Read more about Unit Testing on device in the MAUI docs](https://learn.microsoft.com/en-us/dotnet/maui/deployment/unit-testing?view=net-maui-11.0#run-tests-in-a-platform-app).

# Diagnostics

`EnableCrashReport` enables the .NET crash-report writer for CoreCLR and NativeAOT apps. Consider enabling it in your test and release pipeline, then verify that the resulting reports fit your existing crash-collection process.

# Check list

Here is a a check list for the most important changes on each platform.

## Android

- [ ] Build with JDK 21 or later
- [ ] Update `SupportedOSPlatformVersion` to `24.0`
- [ ] Test Release builds on CoreCLR
- [ ] Test trimming and NativeAOT when the app or its libraries use custom `IJavaObject` types, JNI callbacks, `[Export]`, `[ExportField]`, or older Android bindings.
- [ ] Verify `PublishTrimmed=true` when relying on CoreCLR ReadyToRun; test the startup-size trade-off before enabling full ReadyToRun.
- [ ] Test R8-enabled release builds, especially code that uses Java reflection or name-based member lookup. Set `AndroidR8ObfuscationMode` to `disabled` only as a temporary compatibility fallback.
- [ ] Review HTTP exception and cancellation handling for `AndroidMessageHandler`: failures now surface as `HttpRequestException` and cancellation as `OperationCanceledException`.

## iOS

- [ ] Upgrade to Xcode 27.0 and confirm that the project continues to compile against its updated Apple SDK bindings.
- [ ] Switch to `UISceneDelegate` for deep-links, notifications, user-activities, quick-actions etc.
- [ ] Test CoreCLR on physical devices as well as simulators. Use `UseMonoRuntime=true` only as a short-term debugger or compatibility fallback.
- [ ] Test trimmed Release builds and review custom Objective-C bindings and reflection-heavy dependencies under the trimmable-static registrar.
- [ ] Verify IPA symbol uploads and crash-report collection. Keep the default `IpaIncludeSymbols=true` unless the symbols are deliberately managed elsewhere, and evaluate `EnableCrashReport` for CoreCLR apps.
- [ ] Validate `NSUrlSessionHandler` behavior when using digest authentication, redirects, or custom HTTP proxies, and remove redundant app-extension build steps.

# Read more

The information in this blog post is a digest of the release notes that were published with each .NET 11 release and the source can be found here:

- [.NET 11 Preview 1 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview1/dotnetmaui.md)
- [.NET 11 Preview 2 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview2/dotnetmaui.md)
- [.NET 11 Preview 3 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview3/dotnetmaui.md)
- [.NET 11 Preview 4 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview4/dotnetmaui.md)
- [.NET 11 Preview 5 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview5/dotnetmaui.md)
- [.NET 11 Preview 6 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview6/dotnetmaui.md)
- [.NET 11 Preview 7 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview7/dotnetmaui.md)
- [.NET 11 RC1 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/rc1/dotnetmaui.md)
- [.NET 11 RC2 .NET MAUI release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/rc2/dotnetmaui.md)

