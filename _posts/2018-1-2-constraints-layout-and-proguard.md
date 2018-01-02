---
layout: post
title: Constraint Layout and Proguard
author: Tomasz Cielecki
comments: true
date: 2018-1-2 15:15:00 +0100
tags:
- Xamarin
- Layout
- Proguard
---

I have made an App which uses RelativeLayout and LinearLayout a lot. I wanted to convert some of these layouts to use ConstraintLayout, which in many cases simplifies a layout. Also when using LinearLayout with weights it can improve performance as well.

My App also uses Google Play Services and other libraries which increase the DEX count by a lot. Hence, I need to have multi-dex and Proguard enabled, which can complicate things a bit.

As a reader you may know, Xamarin does not provide Proguard rules which are merged with the default rules, similar to what a native Android dev might be used to from Android Studio, gradle and the ecosystem around that. So you have to add rules yourself to prevent Proguard from removing code that you are actually using.

In the case of ConstraintLayout I added the following rules in my `proguard.cfg` file in the app project, to have it survive Proguard's stripping process.

```
# support constraint
-dontwarn android.support.constraint.**
-keep class android.support.constraint.** { *; }
-keep interface android.support.constraint.** { *; }
-keep public class android.support.constraint.R$* { *; }
```

These rules can be applied to other namespaces that you might find that Proguard is stripping out. Usually this will be reflected in your App, through an exception thrown at runtime telling you some class cannot be found in the Dex List.

You can read more about [Proguard in the Xamarin Android documentation](https://developer.xamarin.com/guides/android/deployment,_testing,_and_metrics/release-prep/proguard/) and how to configure it for your App.