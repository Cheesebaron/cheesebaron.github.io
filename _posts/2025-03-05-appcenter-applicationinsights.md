---
layout: post
title: Migrating AppCenter Analytics Events to Application Insights
author: Tomasz Cielecki
comments: true
date: 2025-03-05 16:00:00 +0100
tags:
- AppCenter
- Azure
- dotnet
---
With [AppCenter][appcenter] closing 31st of March, I bet some people are scrambling to find out what to do instead.
In my organization we've moved crashes into [Sentry][sentry]. However, there is still the question of what to do about
Analytics events, which [Sentry][sentry] does not have an offering for.

We had AppCenter analytics events being exported into [Application Insights][appinsights]. To patch this functionality,
one could add not too many lines of code to replicate more or less what AppCenter exported.

In your Mobile App, you can add the [`Microsoft.ApplicationInsights` NuGet package][nuget] and create a `TelemetryClient` like so:

```csharp
var config = new TelemetryConfiguration
{
    ConnectionString = MyConnectionString
};
telemetryClient = new TelemetryClient(config);
```

To set similar global properties on events which AppCenter did you add these properties:

```csharp
telemetryClient.Context.Device.Id = your device id (you have to come up with something for this, I just save a GUID);
telemetryClient.Context.Device.OperatingSystem = GetOperatingSystem();
telemetryClient.Context.Device.Model = Microsoft.Maui.Devices.DeviceInfo.Manufacturer;
telemetryClient.Context.Device.Type = Microsoft.Maui.Devices.DeviceInfo.Model;
telemetryClient.Context.GlobalProperties.Add("AppBuild", Microsoft.Maui.ApplicationModel.VersionTracking.CurrentBuild);
telemetryClient.Context.GlobalProperties.Add("AppNamespace", Microsoft.Maui.ApplicationModel.AppInfo.PackageName);
telemetryClient.Context.GlobalProperties.Add("OsName", Microsoft.Maui.Devices.DeviceInfo.Platform.ToString());
telemetryClient.Context.GlobalProperties.Add("OsVersion", Microsoft.Maui.Devices.DeviceInfo.VersionString);
telemetryClient.Context.GlobalProperties.Add("OsBuild", Microsoft.Maui.Devices.DeviceInfo.Version.Build.ToString());
telemetryClient.Context.GlobalProperties.Add("ScreenSize",
    $"{Microsoft.Maui.Devices.DeviceDisplay.MainDisplayInfo.Width}x{Microsoft.Maui.Devices.DeviceDisplay.MainDisplayInfo.Height}");

static string GetOperatingSystem()
{
    var os = Microsoft.Maui.Devices.DeviceInfo.Platform.ToString();
    var version = Microsoft.Maui.Devices.DeviceInfo.VersionString;
    return $"{os} ({version})";
}
```

If you want to identify users you can do:

```csharp
telemetryClient.Context.User.Id = userId;
telemetryClient.Context.GlobalProperties["UserId"] = userId;
```

Then to track events you can do:

```csharp
public void TrackEvent(string eventName, IDictionary<string, string> properties = null)
{
    if (properties != null)
    {
        // AppCenter exported event properties in a nested way
        telemetryClient.TrackEvent(eventName,
            new Dictionary<string, string> { { "Properties", ToPropertiesValue(properties) } });
    }
    else
    {
        telemetryClient.TrackEvent(eventName);
    }
}

private static string ToPropertiesValue(IDictionary<string, string> dictionary) =>
    "{" + string.Join(",", dictionary.Select(kv => $"\"{kv.Key}\":\"{kv.Value}\"")) + "}";
```

This will get you most of the way and any dashboards you've made based on the data in `customDimensions.Properties` can
be kept alive indefinitely or until you switch to something else.

[appcenter]: https://appcenter.ms
[sentry]: https://sentry.io
[appinsights]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview
[nuget]: https://www.nuget.org/packages/Microsoft.ApplicationInsights
