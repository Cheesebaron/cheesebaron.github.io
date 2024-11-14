---
layout: post
title: .NET 9 Gotchas!
author: Tomasz Cielecki
comments: true
date: 2024-11-14 17:15:00 +0100
tags:
- dotnet
- Android
- iOS
---

[.NET 9 was just released][net9announcement] and it has brought with it a lot of cool new performance features. Some that might do a lot of good things. Some of them might cause you a little bit of a headache.

After the announcement I dove straight into upgrading stuff to .NET 9 and I have found a few things that tripped up the smoothness of the experience a bit. However, nothing that forced me to abandon all hope and stick to .NET 8. The experience overall was pretty good. Simply updating TFM from `net8.0-{ios|android}` to `net9.0-{ios|android}` and updating a few `Microsoft.*` and `System.*` packages I referenced to the new and shiny `9.0.0` versions on NuGet.

## So what are the issues then?

So a few things you might want to know, to help you along.

### iOS Objective C Registrar

On iOS as announced in their [Release notes Wiki page][iosreleasenotes], there is a new Objective C registrar, to better support NativeAOT. This may cause you issues with binding libraries, which for instance I found with the package ImageCaching.Nuke which I use and maintain. That at runtime, with the new registrar, I would get this nice runtime exception as soon as I would call in to the library:

```log
ObjCRuntime.RuntimeException: Could not find the type 'ObjCRuntime.__Registrar__' in the assembly 'ImageCaching.Nuke'.
  ?, in MapInfo RegistrarHelper.GetMapEntry(string) x 2
  ?, in Type RegistrarHelper.LookupRegisteredType(Assembly, uint)
  ?, in MemberInfo Class.ResolveToken(Assembly, Module, uint)
  ?, in MemberInfo Class.ResolveFullTokenReference(uint)
  ?, in MemberInfo Class.ResolveTokenReference(uint, uint)
  ?, in Type Class.ResolveTypeTokenReference(uint)
  ?, in Type Class.FindType(NativeHandle, out bool)
  ?, in Type Class.Lookup(IntPtr, bool) x 2
  ?, in ImagePipeline Runtime.GetNSObject<ImagePipeline>(IntPtr, IntPtr, RuntimeMethodHandle, bool) x 3
```

Not super nice. But, as the release notes say, you can [opt-out of the new registrar][opt-out] by adding this to your App's csproj file:

```xml
<Target Name="SelectStaticRegistrar" AfterTargets="SelectRegistrar">
    <PropertyGroup Condition="'$(Registrar)' == 'managed-static'">
        <Registrar>static</Registrar>
    </PropertyGroup>
</Target>
```

This fixes the issue and iOS is happy again. Sweet!

### Android Default Runtime Identifiers

On Android you should know that the default Runtime Identifiers (RID) now do not include 32-bit stuff. So if you have an App that you want to run on an older Android device, you may want to include it your self.

In your csproj you can specify the RIDs with:

```xml
<PropertyGroup>
    <RuntimeIdentifiers>android-arm;android-arm64</RuntimeIdentifiers>
</PropertyGroup>
```

There are also `android-x86` and `android-x64` you can specify if you are interested in those.

### Android API 35 16KB Page Sizes

When targeting Android 35, which .NET9 does out of the box, you will now get warnings like this for native libraries that have not yet been recompiled with the latest tooling:

```log
Warning XA0141 : NuGet package '<unknown>' version '<unknown>' contains a shared library 'libsentry.so' which is not correctly aligned. See https://developer.android.com/guide/practices/page-sizes for more details
```

This is a [new thing in Android 15 (API 35), which is for new devices to optimize memory performance][16kbpages]. Hopefully Sentry among other libraries rebuild with new tooling soon and add fixes these warnings.

Apart from this I haven't found any issues so far. I will update the post if anything else comes up.

Also just as a reminder, the libraries you consume in your .NET 9 App, do not need to target .NET 9, so if you are consuming a .NET 6/7/8 package that is OK!

[net9announcement]: https://devblogs.microsoft.com/dotnet/announcing-dotnet-9/
[iosreleasenotes]: https://github.com/xamarin/xamarin-macios/wiki/.NET-9-release-notes
[opt-out]: https://github.com/xamarin/xamarin-macios/wiki/.NET-9-release-notes#opting-out
[16kbpages]: https://developer.android.com/guide/practices/page-sizes
