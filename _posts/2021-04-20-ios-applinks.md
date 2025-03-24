---
layout: post
title: Universal AppLinks on iOS
author: Tomasz Cielecki
comments: true
date: 2021-04-20 16:00:00 +0100
tags:
- Xamarin
- Xamarin.iOS
- AppLinks
- dotnet
---

I was lucky to be tasked to get Universal AppLinks working, for a cool feature in the TrackMan Golf App I work on at work. I thought this would be super easy to do. But, oh how naive and wrong I was.

So in essence, the App is supposed to open if I hit a URL with a specific pattern for one of our domains. Something like: `https://trackman.com/pin/123456`. Where the numbers can differ.

I've done this before with a custom scheme like `trackman://pin/123456`, which is just a simple entry in the Info.plist for your App. However, for Universal AppLinks, there are many more moving parts involved.

## 1. Associate your App(s) to a specific domain
I guess, in order to not be able to open arbitrary URLs in your App. Apple, requires you to host a file called `apple-app-site-association` on your Web Server. This file should not have any file extensions. Make sure you also serve it with the `Content-Type: application/json`. Also, there should be no redirects to get to this URL.

This file should be in either the root or in a `.well-known` folder. For instance:

- `https://trackman.com/apple-app-site-association`
- `https://trackman.com/.well-known/apple-app-site-association`

Apple will check the `.well-known` folder first, then fall back to the root.

This check happens when you install the App on your device. There is a process called `swcd` that will run and check these associasions on your device during install. If any of this stuff fails, this `swcd` process is what you look for in the device log.
The errors will look something like this.

```
Request for '<private>' for task AASA-80AD262A-3EF6-42A2-B992-AC97234187647 { domain: *.tr....com, bytes: 0, route: cdn } denied because the CDN told us to stop with HTTP Status...
```

You will need to deduct yourself which domain it relates to, because the console entries redact some of the values. However, it will tell you what is wrong. Some common issues are:
- SSL certificate doesn't match the domain entry you've registered
- Apple CDN cannot access the file
- The JSON is in a wrong format

Yes, you read it right. From iOS 14 and up, Apple will hit the file through their CDN. So you have to make sure their CDN can reach the file.

The contents of this JSON file will look as follows. Also refer to [Listing 6-1 in these Apple docs.][appledocs] for more details of the format.

```json
{
    "applinks": {
        "apps": [],
        "details": [
            {
                "appID": "<app id prefix>.<bundle id>",
                "paths": [ "/pin/*"]
            }
        ]
    }
}
```

The JSON file format that they suggest in the [Supporting Associated Domains documentation][domains], did not work for me! If something doesn't work, try swiching between the two. Unfortunately Apple does not provide a JSON-Schema for this, so it is not easy to validate if you entered stuff correctly, except that you are returning valid JSON.

The App id prefix and bundle name can be located in the developer portal where you have created your App in the [Identifiers section][identifiers]. The App Id will be the same as your Team ID usually. Bundle id, is the Id of your App that you have registered in there. So `appID` that goes into the JSON file will look something like: `2V9BB354QZ.my.awesome.app`.

The `paths` is an array of paths. So anything that goes after your domain. You can use wildcards like:
- `*` will match any substring
- `?` will match a _single_ character

Examples:
- `/tamarin` will match a specific path
- `/cats/*` will match everything in the "cats" section of your website
- `/cats/archives/202?` will match everything in "cats/archives" and 2020-2029

You can combine the wildcard characters. The `paths` array can have multiple, comma delmited paths for a App Id.

To exclude a path you can write `NOT /cats/*`.

## 2. Enabling the Associated Domain entitlement in provisioning profiles
Go to Certificates, Identifiers & Profiles in the Apple Developer Portal. Here you will need to enable the "Associated Domains" entitlement for your App.

In the list of Identifiers find your App and Edit it. Enable the "Associated Domains" entitlement
[![Screenshot of Associated Domains option in Developer Portal][bundleid]][bundleid]

Now go to any Provisioning profiles you have an regenerate them. You might be using Fastlane or something like that. Read the docs for your tool to download or regnerate provisioning profiles.

After regenerating the Provisioning profiles, you might need to update any CI pipeline and machines that you build on with these new profiles.

## 3. Adding Associated Domains to your `Entitlements.plist` files
If you don't have a Entitlements.plist file, it is now time to create one. [Microsoft provides good docs for you to read more about how to create one][entitlementsdoc]. You will need to add all the domains you wish your Apps to open. These of course need to match the domains you added in the first step in the `apple-app-site-association` file.
This will look something like:
```xml
<key>com.apple.developer.associated-domains</key>
<array>
    <string>applinks:trackman.com</string>
    <string>applinks:ostebaronen.dk</string>
    <string>applinks:sub-domain.ostebaronen.dk</string>
</array>
```

You can also use wildcards here:
```xml
<string>applinks:*.ostebaronen.dk</string>
```

This will match any sub-domain, but not include the raw `ostebaronen.dk` domain. If you can avoid it, I recommend not to use the wildcard here. I had issues with the Apple CDN not being able to process the `apple-app-site-association` file, since it seems like it expects a wildcard entry in the SSL certificate too. I did not have that. If you are going to use it, make sure to test this thoroughly.

## 4. Extending `AppDelegate.cs` to handle the URL requests to your App
Luckily it seems like how our App is triggered is that first it launches the App fully. So `FinishedLaunching` is allowed to complete. Then `ContinueUserActivity` gets triggered with the URL. For good measures I also added an override for `UserActivityUpdated`. So in your `AppDelegate.cs` override these two methods.

```csharp
public override bool ContinueUserActivity(UIApplication application, NSUserActivity userActivity, UIApplicationRestorationHandler completionHandler)
{
    CheckForAppLink(userActivity);
    return true;
}

public override void UserActivityUpdated(UIApplication application, NSUserActivity userActivity)
{
    CheckForAppLink(userActivity);
}
```

To get the URL from the `NSUserActivity` I got inspired from the `AppDelegate.cs` that Xamarin.Forms provides.

The code looks like:

```csharp
private void CheckForAppLink(NSUserActivity userActivity)
{
    var strLink = string.Empty;
    if (userActivity.ActivityType == "NSUserActivityTypeBrowsingWeb")
    { 
        strLink = userActivity.WebPageUrl.AbsoluteString;
                
    }
    else if (userActivity.UserInfo.ContainsKey(new NSString("link")))
    {
        strLink = userActivity.UserInfo[new NSString("link")].ToString();
    }

    // do something clever with your URL here
    NavigateToUrl(strLink);
}
```

## 5. Debugging all of this
This part is a huge pain. First of all, it appears to me, when you deploy an Application using Xcode or Visual Studio, the `swcd` process doesn't run and check the `apple-app-site-association` file. This means when you hit your URL in Safari or through a QR code, it will _not_ suggest to open your App.
I had to grab builds from my CI pipeline for this to work 😢.

But things to check:
- `apple-app-site-association` is reachable
- the URL you are hitting is not returning 404 (although it might actually still work)
- entries in your `Entitlements.plist` match the association json file
- if you have multiple Entitlements for different configurations, make sure you are using the correct one
- check the console output on your device
  - Xcode -> Window -> Devices & Simulators -> Devices -> Select device -> Open Console
  - Click Errors and Faults
  - add `swcd` in the search field

## 6. Bonus: Handling custom schemes
If you also need to handle custom URL schemes. The only thing you need to do is to register the schemes in your `Info.plist` file:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>my.bundle.id</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>awesomescheme</string>
            <string>tamarin</string>
        </array>
    </dict>
</array>
```

Then override `OpenUrl` in your `AppDelegate.cs` file:

```csharp
public override bool OpenUrl(UIApplication app, NSUrl url, NSDictionary options)
{
    if (url.ToString().Contains("/my-pattern"))
    {
        NavigateToUrl(url.ToString());
        return true;
    }

    return base.OpenUrl(app, url, options);
}
```

[appledocs]: https://developer.apple.com/library/archive/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW1
[domains]: https://developer.apple.com/documentation/safariservices/supporting_associated_domains?language=objc
[identifiers]: https://developer.apple.com/account/resources/identifiers/list
[bundleid]: {{ site.url }}/assets/images/ios-applinks/bundle.png "Screenshot of Associated Domains option in Developer Portal"
[entitlementsdoc]: https://docs.microsoft.com/en-us/xamarin/ios/deploy-test/provisioning/entitlements "Working with entitlements"
