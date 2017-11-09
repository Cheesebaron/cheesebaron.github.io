---
layout: post
title: Modifying XML nodes with Cake
author: Tomasz Cielecki
comments: true
tags:
- Cake
- Xamarin
---

I am using Cake everywhere I can for my builds. It is such a nice way to define your builds in a C# DSL and run them on any environment.

Recently I had the need to patch a bunch of attributes in some XML files. Of course Cake has a tool for that! This is called `XmlPoke`. This tool along with a little bit of XPath knowledge and you are ready to update your XML files!

Lets say you want to update your AndroidManifest.xml with the newest versions for your App. This is usually done in the root element of the manifest, funnily enough called `manifest`:

```csharp
<manifest 
    xmlns:android="http://schemas.android.com/apk/res/android"
    package="dk.ostebaronen.cheesyapp"
    android:versionCode="1"
    android:versionName="1.2.3"
```

The two attributes `android:versionCode` and `android:versionName` are the ones you want to update.

So first thing needed to be done is to load the namespaces. Even if you have a namespace looking like `xmlns="http://some.schema.com/"` it needs to be loaded and used in the XPath. To load these `XmlPoke`
has a `XmlPokeSettings` class:

```csharp
var settings = new XmlPokeSettings
{
    Namespaces = new Dictionary<string, string> 
    {
        {"android", "http://schemas.android.com/apk/res/android"}
    }
};
```

In the case of the Android Manifest, we only have the `android` namespace. In the `Namespaces` dictionary, you would otherwise load all the namespaces that you need for your XPath queries. In the case of a naked `xmlns` you would also name it something, it could be called `root`.

With the settings in place we are ready to modify the Android Manifest with the `XmlPoke` alias. The XPath for the two attributes would be, `/manifest/@android:versionCode` and `/manifest/@android:versionName`. Usage in Cake would be as follows:

```csharp
XmlPoke(manifestFilePath, "/manifest/@android:versionCode",
    versionCode, settings);

XmlPoke(manifestFilePath, "/manifest/@android:versionName",
    versionName, settings);
```

Tying all this information together in a Cake `Task`:

```csharp
Task("Update-Android-Manifest-Version")
    .Does(() => 
{
    var versionCode = "2";
    var versionName = "1.2.4";

    var settings = new XmlPokeSettings
    {
        Namespaces = new Dictionary<string, string> 
        {
            {"android", "http://schemas.android.com/apk/res/android"}
        }
    };

    XmlPoke(manifestFilePath, "/manifest/@android:versionCode", 
        versionCode, settings);
    XmlPoke(manifestFilePath, "/manifest/@android:versionName", 
        versionName, settings);
});
```

For `versionCode` and `versionName` I like to use GitVersion to calculate these, but you could load these from somewhere else, or they could be arguments to your Cake script, it doesn't really matter as long as they are strings.

Lets take a look at how this looks when you have a naked `xmlns` declaration. Here is what the root looks like for a Azure Service Fabric ApplicationManifest.xml file:

```xml
<ApplicationManifest 
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    ApplicationTypeName="CheesyApp"
    ApplicationTypeVersion="1.2.3"
    xmlns="http://schemas.microsoft.com/2011/01/fabric">
```

In this case we still need some namespace declarations:

```csharp
var settings = new XmlPokeSettings
{
    Namespaces = new Dictionary<string, string> 
    {
        {"sf", "http://schemas.microsoft.com/2011/01/fabric"}
    }
};
```

`xsi` and `xsd` are not super important, unless you are going to query nodes using these later on. But to update the `ApplicationTypeVersion` we need the naked namespace.

The query in this case would look like `/sf:ApplicationManifest/@ApplicationTypeVersion`. These Service Fabric Application Manifests also have some `ServiceManifestRef` nodes with `ServiceManifestVersion` for each of the services you have in your app. 

```xml
<ApplicationManifest ApplicationTypeVersion="1.2.3"
    xmlns="http://schemas.microsoft.com/2011/01/fabric">
    <Parameters>
        <Parameter />
        <Parameter />
        <Parameter />
    </Parameter>

    <ServiceManifestImport>
        <ServiceManifestRef ServiceManifestVersion="1.2.3">
    </ServiceManifestImport>

    <ServiceManifestImport>
        <ServiceManifestRef ServiceManifestVersion="1.2.3">
    </ServiceManifestImport>

    ...
```

For the scenario here with multiple attributes XPath can also help you. The query for the `ServiceManifestVersion` would look like `//sf:ServiceManifestRef/@ServiceManifestVersion`. Notice the two slashes `//`. In XPath this means select multiple nodes.

Hopefully this gave you a good idea about the usage of XmlPoke in Cake!