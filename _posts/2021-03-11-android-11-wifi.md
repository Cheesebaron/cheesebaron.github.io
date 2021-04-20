---
layout: post
title: New WiFi API in Android 11
author: Tomasz Cielecki
comments: true
date: 2021-03-11 9:00:00 +0200
tags:
- Xamarin
- Xamarin.Android
- Network
- WiFi
---

Android 10 messed up royally with removing the API to add networks and connect to it on a users device. The API was removed and as an alternative they gave us:
1. A suggestion API, which shows a low priority notification, suggesting to connect to a given network. This notification would need to be swiped down a couple of times to reveal the YES/NO options. Then the device might choose not to connect to it anyways.
2. A way to connect to a local network, not providing any Internet connectivity, through a System dialog appearing in the App. This dialog took forever to find your network. However, this method was not very useful for anyone as no Internet on this connection.

The last alternative, but technically not a part of the API for connecting to WiFi, would be to show a settings panel, where the user could _manually_ enter credentials for a WiFi network in the list.

## New Android 11 API

> Note: this code will throw a `ActivityNotFoundException` if run on older Android versions. Make sure to add some version checking.

I guess the folks at Google got a bit backlash or they somehow changed their mind. However, they've now added an Intent you can fire to add a Network on the device.

The code for this looks something like this:

```csharp
var intent = new Intent(
    "android.settings.WIFI_ADD_NETWORKS");
var bundle = new Bundle();
bundle.PutParcelableArrayList(
    "android.provider.extra.WIFI_NETWORK_LIST",
    new List<IParcelable>
    {
        new WifiNetworkSuggestion.Builder()
            .SetSsid(ssid)
            .SetWpa2Passphrase(password)
            .Build()
    });

intent.PutExtras(bundle);

StartActivityForResult(intent, AddWifiSettingsRequestCode);
```

> Note: `AddWifiSettingsRequestCode` is just an Integer you define.

Then in `OnActivityResult` you can figure out whether any network in the list you provided was added with:

```csharp
if (requestCode == AddWifiSettingsRequestCode)
{
    if (data != null && data.HasExtra(
        "android.provider.extra.WIFI_NETWORK_RESULT_LIST"))
    {
        var extras =
            data.GetIntegerArrayListExtra(
                "android.provider.extra.WIFI_NETWORK_RESULT_LIST")
                ?.Select(i => i.IntValue()).ToArray() ?? new int[0];

        if (extras.Length > 0)
        {
            var ok = extras
                .Select(GetResultFromCode)
                .All(r => r == Result.Ok);
            // if ok is true, BINGO!
            return;
        }
    }
}
```

The extras will return a list of `Result` which indicate whether they were added if it is `Result.OK` or not if it is `Result.Cancel`.

GetResultFromCode, simply parses the integers returned and turns them into a `Result`:

```csharp
private static Result GetResultFromCode(int code) =>
    code switch
    {
        0 => Result.Ok, // newly added
        2 => Result.Ok, // wifi already there
        _ => Result.Canceled
    };
```

Running this code will show you a dialog looking something like this

[![Screenshot of Save this network dialog][dialog]][dialog]

> Note: This code works and runs fine on a Google Pixel 3a XL running latest Android 11. However, on my OnePlus 8 running OP8_O2_BETA_3 opening the intent fails, because of OnePlus's Settings App does not implement the AppCompat theme, crashing. I've reported this issue to OnePlus but never heard back from them.

You can check out the code from my [Android 11 WiFi Repository on GitHub](https://github.com/Cheesebaron/Android11WiFi) and have a go testing it yourself.

[dialog]: {{ site.url }}/assets/images/android11wifi/dialog.png "Screenshot of Save this network dialog"