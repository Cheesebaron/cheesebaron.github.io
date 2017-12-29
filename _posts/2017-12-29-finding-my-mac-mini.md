---
layout: post
title: Finding My Mac Mini On The Network
author: Tomasz Cielecki
comments: true
date: 2017-12-29 17:00:00 +0100
tags:
- Xamarin
- Networking
---

Often when I reboot my Mac Mini or it has been off for a long while, I find myself not being able to figure out which IP it has, when it is up and running again. I need this IP to connect to it through VNC from my Windows PC when I want to do stuff on it.

Now the Xamarin integration into Visual Studio (VS) can see the Mac. However, I really don't want to spin up VS just to figure out the IP of my Mac. So question is... How does VS find my Mac? Easy! Using something called Bonjour. A lot of modern devices and Mac's use Bonjour to advertise themselves on the network, along with the services they provide. In the Mac case this is often used by iTunes, AirPlay and other services to find devices to play to and from.

So by utilizing Bonjour I can find my Mac easily. For this I've created a little LinqPad script to help me.

I am using the [Zeroconf](https://www.nuget.org/packages/Zeroconf/) NuGet by [Oren Novotny](https://oren.codes/) to get a nice and easy to use client for discovering devices.

So first thing I do is to enumerate all the devices and their services on the network, which is done simply with.

```csharp
private async Task<IReadOnlyList<IZeroconfHost>> EnumerateAllServicesFromAllHosts()
{
    // get all domains
    var domains = await ZeroconfResolver.BrowseDomainsAsync();

    // get all hosts
    return await ZeroconfResolver.ResolveAsync(domains.Select(g => g.Key));
}
```

Going through these I can simply look for a Host with the `DisplayName` of my Mac. Mine is called `macmini-tomasz`.

```csharp
var services = await EnumerateAllServicesFromAllHosts();
var host = services.FirstOrDefault(s => s.DisplayName == "macmini-tomasz");
```

Now that I have the actual host it has a property `IPAddress` which I can use for my VNC client!

Voila! With these simple steps you can find your device on the network, which broadcasts using Bonjour.