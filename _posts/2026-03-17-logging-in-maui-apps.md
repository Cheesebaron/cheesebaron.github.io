---
layout: post
title: Logging in .NET MAUI Apps
author: Tomasz Cielecki
comments: true
date: 2026-03-17 17:00:00 +0100
tags:
- Android
- iOS
- dotnet
- devops
- MAUI
- Sentry
---

A topic I have seen come up again multiple times and multiple answers for is how to add logging to your App. People do it slightly differently, but in the end logs end up in files and logging systems.

Logging goes really well hand in hand with your crash reporting system, to have a better idea of what happened before the App crashed. So systems like Sentry and Firebase support enriching crashes with logs. I highly recommend integrating with these as it removes a lot of guesswork if done well.

So let us dig into how I approach logging in a lot of Apps I work on.

# Microsoft Extensions Logging

Most modern Apps, at least when dealing with .NET, will have a Inversion of Control (IoC) container set up. Whether it is Splat, Microsoft Extensions Dependency Injection (MEDI), MvvmCross, Autofac or something else. Microsoft Extensions Logging (MEL) plays really well into this.

> The `Microsoft.Extensions.Logging` package assumes you are using `Microsoft.Extensions.DependencyInjection`, if your setup is not using that, there may be some more setup to do. I will show an example of how I do it in MvvmCross

The [`Microsoft.Extensions.Logging.Abstractions`][loa] package provides abstractions for you to use around your Application. Such as `ILogger`, which is the logger you would inject to your classes, or alternatively a `ILoggerFactory` which you can use to create a new instance of a `ILogger`.

The [`ILogger` interface][iloggerdoc], provides the `Log()` method, which you can use or one of the multitudes of extension methods to provide severity of your log entry along with message, exceptions and parameters.

## Adding Logging in your IoC container

If you are using MEDI the setup is simple.

Add the MEL NuGet package:

```sh
dotnet package add Microsoft.Extensions.Logging
```

Then on your `IServiceCollection` when building it add:

```csharp
serviceCollection.AddLogging();
```

This will add a bunch of dependencies to your container among these:

- `ILoggerProvider`
- `ILoggerFactory`
- `ILogger<T>`

This way you can resolve any of these in your classes. Typically you would use the generic `ILogger<T>` like:

```csharp
class MyClass(ILogger<MyClass> logger)
```

### Splat and MvvmCross

Splat have their own way to add MEL to their IoC container through the package `Splat.Microsoft.Extensions.Logging`. You have to provide a `ILoggerProvider` when using the package.

For MvvmCross per default it opts into using MEL abstractions and you have to provide a `ILoggerProvider` and `ILoggerFactory` in the `Setup.cs` file.

## Serilog

`Microsoft.Extensions.Logging` does not provide much in terms of where logs end up. Out of the box it comes with a Console and Debug logger you can configure. For logging to other places (sinks) I use [Serilog, which is a logging library that from ground up is designed with structured data in mind][serilog].
There is also a lot of packages that build on top of Serilog which provide sinks to log to files, Android log, iOS log, Sentry, seq, fluentd and many other systems. And if a sink doesn't exist, it is very easy to provide your own code to sink somewhere specific.

[Serilog plays well with MEL and provides a package to integrate with it][serilog-mel]. You can install with:

```sh
dotnet package add Serilog.Extensions.Logging
```

This package provides `SerilogLoggerProvider` and `SerilogLoggerFactory` which you can pass to Splat, MvvmCross and other integrations. Or if you are already using Microsoft Extensions Dependency Injection and Microsoft Extensions Logging it is a matter of adding the following line to your logging builder:

```csharp
loggingBuilder.AddSerilog();
```

This will set up Logger Provider to send logs into the Serilog logger along with any other logger providers you configure MEL with.

You still need to set up a Serilog logger configuration to build the Serilog logger.

I often do something like:

```csharp
var outputTemplate = "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} {SourceContext:l}:#{ThreadId} [{Level}] {Message}{NewLine}{Exception}";

Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .Enrich.WithThreadId()

    // Serilog.Sinks.Async
    .WriteTo.Async(a =>
        // Serilog.Sinks.File
        a.File(
            // Serilog.Formatting.Compact
            new CompactJsonFormatter(),
            Path.Combine(logFolder, "log.clef"),
            LogEventLevel.Information,
            7_000_000,
            rollingInterval: RollingInterval.Day,
            rollOnFileSizeLimit: true,
            buffered: true,
            retainedFileCountLimit: 14,
            flushToDiskInterval: TimeSpan.FromSeconds(5))
    )
    .WriteTo.Async(a =>
        // Serilog.Sinks.Trace
        a.Trace(
            LogEventLevel.Verbose,
            outputTemplate)
    )
    .WriteTo.Async(a => 
        // custom sink
        AndroidLogSink.LoggerConfigurationAndroidExtensions.AndroidLog(a,
            LogEventLevel.Information,
            outputTemplate)
    )
    .CreateLogger();
```

This requires some extra packages such as:

- [`Serilog.Enrichers.Thread`][set]
    - Enriches log entries with thread Id to know which thread we called from
- [`Serilog.Extensions.Logging`][sel]
    - Package providing integration with Microsoft.Extensions.Logging
- [`Serilog.Formatting.Compact`][sfc]
    - `clef` compact logging format to preserve structured log parameters and message templates
- [`Serilog.Sinks.Async`][ssa]
    - Sink to asynchronously write to sinks, this can help performance issues in resource constrained environments, using `WriteTo.Async`.
- [`Serilog.Sinks.File`][ssf]
    - Sink to allow writing to a file, with options to roll the logs when file exceeds a certain size, when date changes etc. Also writing buffered can help performance issues at a risk of losing some log entries, rather than having to do disk I/O for each log call
- [`Serilog.Sinks.Trace`][sst]
    - Sink to sink into IDE output window

This shows that with a few lines of code you have a lot of control of where to output logs with Serilog. You can likely do something similar with other logging frameworks, I just like how Serilog works.

As an alternative to the coded configuration above, Serilog also supports having the configuration defined in a configuration file.

Also having logs as files, it will be easier to export and share log files from your App for QAs and beta testers.
You will need to consider where your `logFolder` will be. Using a folder like `FileSystem.AppDataDirectory` from MAUI Essentials is a safe default and will be private to the App.

## Sentry logs

Sentry has supported adding Logs as breadcrumbs for a very long time. Since Sentry SDK 5.14.0 they've added a `EnableLogs` property, that allows you to send log entries to their Log feature. Not enabling this will still enrich
events with Logs as breadcrumbs.

If you are using `Sentry.MAUI` or `Sentry.Extensions.Logging` you can add Sentry in the mix on your logging builder with:

```csharp
loggingBuilder.AddSentry(sentryOptions => 
{
    // initialize sentry setting DSN and other options here

    // send logs to Sentry Logs (false will only add logs to breadcrumbs)
    sentryOptions.EnableLogs = true; 
});
```

> Note: If you are initializing sentry elsewhere using `SentrySdk.Init` make sure to set `options.InitializeSdk = false;` in the logging builder options, otherwise it will initialize a new instance.

## Structured Logging

What is really great about Microsoft Extensions Logging, Serilog and Sentry Logs is that they all support using structured logging. What structured logging boils down to, is to have a logging message template with named parameters and a set of parameters, which make up each log entry. This means when you are filtering log messages, you can filter them by messages, parameter names and parameter values.

For example let us say you have the following logging template:

```csharp
"User changed {SettingName} to {SettingValue}"
```

And when you log use:

```csharp
logger.LogInformation(
    "User changed {SettingName} to {SettingValue}",
    settingName,
    settingValue
);
```

Then when trawling through logs it is easy for you to search for the parameter name `SettingName` but also the actual value of that parameter, rather than only having the realized log message i.e. `"User changed WiFi to false"`. This also makes it easier to set up alerting rules based on log messages etc.
It also allows you to add parameters that are not necessarily in the log message template to enrich a log entry.

This is also where the `clef` format described in the Serilog section above comes into the mix. It stores each log entry with all the context. So you don't lose any of that information, when saving to files.

## Filtering

Sometimes you want to ignore some logs produced by certain contexts in your App. I.e. if you have a third party library, which also hooks into MEL and it produces noise with internal logs. You can filter these out fairly easily when configuring your logging builder.

For instance if you are only interested in Error messages from resilience pipelines:

```csharp
loggingBuilder.AddFilter("Polly", LogLevel.Error);
loggingBuilder.AddFilter("Microsoft.Extensions.Resilience", LogLevel.Error);
loggingBuilder.AddFilter("Microsoft.Extensions.Http.Resilience", LogLevel.Error);
```

Just keep in mind that certain Logging Providers, such as Serilog's, one can opt to ignore these filters. So in the case of Serilog. You would need to add filters on the Serilog configuration. Like:

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Polly", LogEventLevel.Error)
    .MinimumLevel.Override("Microsoft.Extensions.Http.Resilience", LogEventLevel.Error)
    .MinimumLevel.Override("Microsoft.Extensions.Resilience", LogEventLevel.Error)
```

## High-performance logging

[Microsoft Extensions Logging supports source generated logging][high-perf] which avoids boxing and string allocations at runtime, which in certain conditions can improve performance.

How this works is to have a extension class like:

```csharp
public static partial class Log
{
    [LoggerMessage(
        Level = LogLevel.Information,
        Message = "User changed {SettingName} to {SettingValue}")
    ]
    static partial void LogSettingChanged(
        ILogger logger, string settingName, string settingValue);
}
```

Then you can use `logger.LogSettingChanged(settingName, settingValue);` when you want to log that message.

## Custom Serilog Sinks

As I showed in my example above I have my own custom sink for logging on Android. Even though the sink `Serilog.Sinks.Xamarin` exists which provides similar functionality, I had some different requirements for my logs.

Creating your own is super simple by implementing `ILogEventSink` from Serilog. In the `Emit` method you would simply translate Serilog log events to however your custom sink expects this.

This can be useful to make your own sinks to sink into Firebase for instance using their `FirebaseCrashlytics.Instance.Log()` to enrich crash events in Firebase with extra information. Similarly to how Sentry does it with breadcrumbs.

# Conclusion

With a relatively small amount of setup you can have a robust logging pipeline in your .NET MAUI App. To summarize:

1. Use **Microsoft Extensions Logging** as your logging abstraction — it integrates well with most IoC containers and is the standard in the .NET ecosystem.
2. Use **Serilog** (or a similar library) to control where your logs end up — files, platform logs, crash reporting systems, or all of the above.
3. Integrate with your **crash reporting system** like Sentry or Firebase to enrich crash reports with logs. This is where logging pays for itself.
4. Use **structured logging** with named parameters in message templates. Your future self will thank you when searching through logs.
5. Use **filtering** to keep noise down and **async sinks** to keep performance acceptable on resource constrained mobile devices.

The key takeaway is that none of these pieces are complicated on their own, but combined they give you much better observability into what your App is doing in the hands of your users. When that next crash report comes in, you'll have the context to understand what led up to it.

Happy logging!

[loa]: https://www.nuget.org/packages/Microsoft.Extensions.Logging.Abstractions
[iloggerdoc]: https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger
[serilog]: https://serilog.net/
[serilog-mel]: https://www.nuget.org/packages/Serilog.Extensions.Logging
[set]: https://www.nuget.org/packages/Serilog.Enrichers.Thread
[sel]: https://www.nuget.org/packages/Serilog.Extensions.Logging
[sfc]: https://www.nuget.org/packages/Serilog.Formatting.Compact
[ssa]: https://www.nuget.org/packages/Serilog.Sinks.Async
[ssf]: https://www.nuget.org/packages/Serilog.Sinks.File
[sst]: https://www.nuget.org/packages/Serilog.Sinks.Trace
[high-perf]: https://learn.microsoft.com/en-us/dotnet/core/extensions/logging/high-performance-logging