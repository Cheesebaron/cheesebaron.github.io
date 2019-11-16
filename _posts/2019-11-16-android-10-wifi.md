---
layout: post
title: Connecting to WiFi in Android 10
author: Tomasz Cielecki
comments: true
date: 2019-11-16 03:30:00 +0200
tags:
- Xamarin
- Xamarin.Android
- Network
---

Android 10 was recently released and it introduces a bunch of changes in terms of Privacy. This means that access to `/proc/net` from the Linux sub-system has been restricted, which requires you to use `NetworkStatsManager` and `ConnectivityManager` to get VPN information.

It adds restrictions to who is allowed to enable/disable WiFi. We could previously use `WifiManager.SetWifiEnabled()`, but not anymore, this method will return `false`. You will need to show one of the new [Settings Panels][sp], which shows a slice of the Android Settings within your App.

What this post will focus on is the restrictions to access to configured networks and connecting to networks. A bunch of the network API has changed, so let us look a bit into what we have available now.

# Suggesting networks
Something new to Android 10 is suggesting networks to connect to. These are just hints to the platform that the networks you provide it, can be connected to and it might decide to chose one of them. When any of these networks are detected nearby, Android will show a notification to the user the first time, which is how they allow connecting to a suggested network.

This could be useful for an Enterprise App, which can allow Access to networks depending on the logged in user or suggest a separate network for guests.

You can suggest networks like so.

```csharp
var guestUsers = new WifiNetworkSuggestion.Builder()
    .SetSsid("GuestNetwork")
    .SetWpa2Passphrase("hunter2")
    .Build();

var secretEnterpriseNetwork = new WifiNetworkSuggestion.Builder()
    .SetSsid("Cyberdyne")
    .SetWpa2Passphrase(":D/-<")
    .Build();

var suggestions = new[] { guestUsers, secretEnterpriseNetwork };

var wifiManager = this.GetSystemService(Context.WifiService) as WifiManager;
var status = wifiManager.AddNetworkSuggestions(suggestions);

if (status == NetworkStatus.SuggestionsSuccess)
{
    // We added suggestions!
}
```

The suggestions you provide can only be added once. If you try add the same suggestion again, the status from `AddNetworkSuggestion` will return `SuggestionsErrorAddDuplicate`. If you need to modify a suggestion, you need to remove it first with  `RemoveNetworkSuggestion`, then add the modified version of it again. Additionally, in order to add these suggestions you will need to add the `CHANGE_WIFI_STATE` permission to your AndroidManifest.xml.

```xml
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
```

> Note: Some of the options on a `WifiNetworkSuggestion` requires you to request Fine Location permission in order to work. Make sure to consult the [Android documentation][suggest_wifi_docs] to be sure.

After you run `AddNetworkSuggestion` don't expect something to happen immediately. There will eventually be a notification in the notification drawer. Which looks like something in the image below. Choosing "Yes" on the notification, won't necessarily automatically connect to the network. However, going to Wifi Settings on your device, it should now know how to connect to that network.

[![suggestion notification][suggest_notif]][suggest_notif][![suggestion wifi settings][suggest_wifi_settings]][suggest_wifi_settings]

# Connecting to Specific Networks
Prior to Android 10, we could explicitly tell Android to connect to a specific Network and it would simply do it. This was done using `WifiManager.AddNetwork()` where you provided a `WifiConfiguration`. Now this API has been deprecated and replaced with `ConnectivityManager.RequestNetwork`, where you build a Network Request with a Wifi Specification, similar to the suggestions I showed you above. However, it also allows you to specify, whether the connection requires Internet, cannot be VPN, has to be WiFi and more.

You will also need to provide a `ConnectivityManager.NetworkCallback`, which will let you know whether connection to the network was successful or not. The idea here is that you are awaiting the requested network to connect and only when that is successful you can continue with your execution.

This is excellent for scenarios, where you need to be on a specific network to do some specific operations. One thing to note is that you will only be able to connect to this network while the App is open. As soon as the App is closed, it disconnects the network.

Let us look at some code.

```csharp
var specifier = new WifiNetworkSpecifier.Builder()
    .SetSsid("cyberdyne")
    .SetWpa2Passphrase("ill be back")
    .Build();

var request = new NetworkRequest.Builder()
    .AddTransportType(TransportType.Wifi) // we want WiFi
    .RemoveCapability(NetCapability.Internet) // Internet not required
    .SetNetworkSpecifier(specifier) // we want _our_ network
    .Build();

var connectivityManager = this.GetSystemService(Context.ConnectivityService) as ConnectivityManager;

connectivityManager.RequestNetwork(request, callback);
```

Here is an example of wanting to connect to the network with SSID "cyberdyne" and passphrase "ill be back". The transport type specifies what kind of network you want to connect to. In this case it is a WiFi network. We do not require any Internet capability on the network. Per default it requests Internet, Not VPN and Trusted [capabilities][capabilities].

The callback looks something like this.

```csharp
private class NetworkCallback : ConnectivityManager.NetworkCallback
{
    public Action<Network> NetworkAvailable { get; set; }

    public override void OnAvailable(Network network)
    {
        base.OnAvailable(network);
        NetworkAvailable?.Invoke(network);
    }
}

var callback = new NetworkCallback
{
    NetworkAvailable = network =>
    {
        // we are connected!
    }
};
```

You can also use `OnUnavailable` to detect that we could not connect to the network in particular, or the user pressed cancel.

[![Connect to Device Request][network_request]][network_request]

If you connect once to a network request, any subsequent requests will automatically connect if you are in range of the network and the user will not have accept it again. Also, while the App is open and you want to disconnect from the requested network, you simply call `ConnectivityManager.UnregisterNetworkCallback()` on the callback.

This hopefully gives you an idea about how to connect to WiFi Networks with Android 10.

You can check out the code from my GitHub repo here: https://github.com/Cheesebaron/Android10Wifi and have a go at playing with it yourself.

[sp]: https://developer.android.com/about/versions/10/features#settings-panels "Android 10 New Features - Settings Panels"
[suggest_notif]: {{ site.url }}/assets/images/android10wifi/suggest_notification.jpg "Screenshot of Wifi Suggestion Notification"
[suggest_wifi_settings]: {{ site.url }}/assets/images/android10wifi/suggest_wifi_settings.jpg "Screenshot of Wifi Settings connected to Suggested Network"
[network_request]: {{ site.url }}/assets/images/android10wifi/network_request.jpg "Screenshot of Dialog Requesting connection to Device"
[suggest_wifi_docs]: https://developer.android.com/reference/android/net/wifi/WifiNetworkSuggestion "WifiNetworkSuggestion Android Documentation"
[capabilities]: https://developer.android.com/reference/android/net/NetworkCapabilities.html "Android Network Capabilities list"