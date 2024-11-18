---
layout: post
title: Central Package Management Transitive Pinning
author: Tomasz Cielecki
comments: true
date: 2024-11-18 15:45:00 +0100
tags:
- dotnet
- nuget
---

I have been using the super nice [Central Package Management feature][cpm], enabled by the `ManagePackageVersionsCentrally` property in many of my .NET projects for quite a while. What it allows you is to define the package versions of the NuGet packages you are referencing in your solution a single `Directory.Packages.props` file in the root of your repo. So instead of having package version scattered around your csproj files, they are not defined a single place and aligned throughout the solution.

So such a `Directory.Packages.Props` file would look something like this:

```xml
<Project>
  <ItemGroup>
    <PackageVersion Include="Microsoft.Extensions.Http" Version="9.0.0" />
  </ItemGroup>
</Project>
```

Then in a projects `.csproj` file you would add:

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Http" />
</ItemGroup>
```

To enable this globally in your solution you would then also have a `Directory.Build.props` file where you add a property like so:

```xml
<Project>
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    </PropertyGroup>
</Project>
```

This is great!

However, a lot of NuGet packages pull in other dependencies that you often don't include in your `Directory.Packages.props` file. With newer versions of .NET and in turn newer versions of NuGet you now have auditing of NuGet packages enabled per default. This will start warning you about packages and with .NET 9, transitive packages, that have security issues when restoring packages. If you have `TreatWarningsAsErrors` enabled, it prevent you from restoring packages and building your project.

Previously if I wanted to use a specific version of a transitive package, I would add it to `Directory.Packages.props` and add it to a project to specify which version of it to use. This is a bit annoying to have to manage as you will need to add it to multiple files.

A colleague of mine made me aware that you can actually [pin transitive packages][transitive-pinning] as well, not only top level packages that you reference directly. Which is a huge help!

If you have ever worked with some of the AndroidX packages, then you will often need to do this as some other package, might be referencing an old version of a transitive package, which isn't directly compatible with a newer top level package you are referencing. Which, at build time will complain about classes from the Java/Kotlin world being specified twice and will fail your build. To fix this you need to pin the transitive packages to a newer version which isn't conflicting. Or, recently we had warnings with `System.Text.Json` version 6.0.0 having a vulnerability warning.

So by setting the `CentralPackageTransitivePinningEnabled` property to `true`, you can now simply add those packages to your `Directory.Packages.props` to pin the version to a specific one, without having to reference it directly in a project.

So your `Directory.Build.props` would then contain something like:

```xml
<Project>
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
        <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
    </PropertyGroup>
</Project>
```

And then I would recommend adding a commented section to your `Directory.Packages.props` like so:

```xml
<Project>
  <!-- packages referenced by projects -->
  <ItemGroup>
    <PackageVersion Include="Microsoft.Extensions.Http" Version="9.0.0" />
  </ItemGroup>

  <!-- transitive pinned packages -->
  <ItemGroup>
    <PackageVersion Include="System.Text.Json" Version="9.0.0" />
    <PackageVersion Include="Xamarin.AndroidX.Activity.Ktx" Version="1.9.3.1" />
    <PackageVersion Include="Xamarin.AndroidX.Collection.Ktx" Version="1.4.5.1" />
    <PackageVersion Include="Xamarin.AndroidX.Lifecycle.Common" Version="2.8.7.1" />
    <PackageVersion Include="Xamarin.AndroidX.Lifecycle.Runtime" Version="2.8.7.1" />
    <PackageVersion Include="Xamarin.AndroidX.Lifecycle.LiveData.Core" Version="2.8.7.1" />
    <PackageVersion Include="Xamarin.AndroidX.Lifecycle.LiveData.Core.Ktx" Version="2.8.7.1" />
  </ItemGroup>
</Project>
```

[cpm]: https://learn.microsoft.com/en-us/nuget/consume-packages/Central-Package-Management
[transitive-pinning]: https://learn.microsoft.com/en-us/nuget/consume-packages/Central-Package-Management#transitive-pinning
