---
layout: post
title: Re-signing a IPA file
author: Tomasz Cielecki
comments: true
date: 2020-02-03 14:00:00 +0200
tags:
- Xamarin
- Xamarin.iOS
---

I just had the task to figure out why we had an App crashing randomly on us all of the sudden. The App is distrubuted through AppCenter only, as a Enterprise build. Turned out that the provisioning profile had expired hence it started to fail.

Now, I did not want to build a new version of the App, since the one distributed and in production is much older and we are in the middle of moving CI for the App into a new enviroment. So the only option was to sign the App again with the new provisioning profile. So here are some notes on how I did that.

The prerequisites for this are:
- IPA file to re-sign
- Distribution certificate is installed in your KeyChain
- mobileprovision file to sign with
- a macOS installation with Xcode installed (I used 11.3.1)

The following snippets are commands you would run in your preferred commandline.

First we need to unizp the IPA file. It will contain a Payload folder with AppName.app inside, where AppName is your App's name. I will use `MyApp` as an example throughout this post. Yours will of course be different.

```
unzip MyApp.ipa
```

You should now have a Payload folder where contents were extracted.

First we need to extract the entitlements for the App, we will need them later when we are signing the App again.

```
codesign -d --entitlements :- "Payload/MyApp.app" > entitlements.plist
```

This will create a `entitlements.plist` file in the folder you are currently in. If you are re-signing the App with a distribution certificate for another team, remember to change identifiers in the entitlements file to match your distribution certificate.

Now before we re-sign the App, we need to remove the old code signing.

```
rm -r Payload/MyApp.app/_CodeSignature
```

We also need to replace the provisioning profile. Assuming your profivisioning profile file name is called `MyApp.mobileprovision` and is located in the same folder we are in.

```
cp MyApp.mobileprovision Payload/MyApp.app/embedded.mobileprovision
```

We should now be ready to re-sign the Application.

```
codesign -f -s "iPhone Distribution: your team name here" --entitlements entitlements.plist Payload/MyApp.app
```

Now we just need to zip the folder and we are ready to distribute it.

```
zip -qr MyApp-resigned.ipa Payload
```

That is it! You are ready to ship your re-signed App!