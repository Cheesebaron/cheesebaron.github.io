---
layout: post
title: Adding Resilience to Refit and your own code
author: Tomasz Cielecki
comments: true
date: 2024-06-25 16:00:00 +0200
tags:
- dotnet
- Android
- iOS
- nuget
- resilience
---

You may be using [Refit][refit] already today in your App or you want to do so. It is a great little REST Api client library where you quickly through interfaces can start communicating with an API and without having to write a bunch of client code yourself.

An example of this looks like so:

```csharp
public interface IMyUserApi
{
    [Get("/users/{userId}")]
    Task<User> GetUser(string userId);
}
```

Then you can use the client like so:

```csharp
var userApi = RestService.For<IMyUserApi>("https://api.myusers.com");
var user = await gitHubApi.GetUser("abcdefg123");
```

Super easy and no need to write any `HttpClient` code to call `GetAsync`.

Refit also nicely integrates with [Microsoft.Extensions.DependencyInjection][di] `IServiceCollection` leveraging [`HttpClientFactory`][httpclientfactory] which most modern Applications should be using. This also allows configuring additional `HttpClientHandler`s to allow more instrumentation and as I later describe, allows configuring some resiliency:

```csharp
services
    .AddRefitClient<IMyUserApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri("https://api.myusers.com"));
```

OK, so with a Refit client registered in the service collection, how can we add some resiliency to it? Perhaps you are already familiar with [`Polly`][polly] directly or using `Microsoft.Extensions.Http.Polly`, this is not recommended anymore, so let me show you how you can set that up using the nice and shiny [`Microsoft.Extensions.Resilience`][mer] and [`Microsoft.Extensions.Http.Resilience`][mehr] packages.

Adding the latter package [`Microsoft.Extensions.Http.Resilience`][mehr] gives you a nice extension method to add a Resilience Handler to your `HttpClient`. So the example above becomes:

```csharp
services
    .AddRefitClient<IMyUserApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri("https://api.myusers.com"))
    .AddStandardResilienceHandler();
```

Just by adding this one line you will get retries set up for you with the [defaults described in the README for the Microsoft.Extensions.Http.Resilience package][defaults] which as of writing this post are:

> - The total request timeout pipeline applies an overall timeout to the execution, ensuring that the request including hedging attempts, does not exceed the configured limit.
> - The retry pipeline retries the request in case the dependency is slow or returns a transient error.
> - The rate limiter pipeline limits the maximum number of requests being send to the dependency.
> - The circuit breaker blocks the execution if too many direct failures or timeouts are detected.
> - The attempt timeout pipeline limits each request attempt duration and throws if its exceeded.

If you want to configure any of these behaviors you can configure that with:

```csharp
.AddStandardResilienceHandler(builder =>
{
    builder.Retry = new HttpRetryStrategyOptions
    {
        MaxRetryAttempts = 2,
        Delay = TimeSpan.FromSeconds(1),
        UseJitter = true,
        BackoffType = DelayBackoffType.Exponential
    };

    builder.TotalRequestTimeout = new HttpTimeoutStrategyOptions { Timeout = TimeSpan.FromSeconds(30) };
});
```

You are fully in control!

If you want to add resiliency to something else, you can also add resilience pipelines directly in your service collection and use the [`Microsoft.Extensions.Resilience`][mer] package:

```csharp
service.AddResiliencePipeline("install-apps", builder =>
{
    builder.AddTimeout(TimeSpan.FromMinutes(3));
});
```

Then resolve and use it with a construction looking something like this:

```csharp
public sealed class AppInstaller(
    ILogger<AppInstaller> logger,
    ResiliencePipelineProvider<string> pipelineProvider)
{
    private readonly ILogger<AppInstaller> _logger = logger;
    private readonly ResiliencePipeline _pipeline = pipelineProvider.GetPipeline("install-apps");

    public async Task<InstallStatus> InstallApp(string downloadPath, CancellationToken cancellationToken)
    {
        // Get context for cancellation and passing along state
        ResilienceContext? context = ResilienceContextPool.Shared.Get(cancellationToken);

        // Execute async method passing state and cancellation token
        Outcome<InstallStatus> outcome = await _pipeline.ExecuteOutcomeAsync(
            async (ctx, state) =>
                Outcome.FromResult(
                    await InstallAppInternal(state, ctx.CancellationToken).ConfigureAwait(false)
                ),
                context,
                downloadPath)
            .ConfigureAwait(false);
        
        // Handle errors from outcome
        if (outcome.Exception != null)
        {
            _logger.LogError(outcome.Exception, "Something went wrong installing app from {DownloadPath}", downloadPath);
        }

        return outcome.Result ?? InstallStatus.Failed;
    }

    // more code here...
}
```

Hope this helps a bit understanding how the resilience libraries work, but just like Polly you can use it with anything you want to retry, with some nice defaults for HTTP requests.

[refit]: https://github.com/reactiveui/refit "Refit automatic type-safe REST client library"
[di]: https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection "Microsoft Dependency Injection docs"
[httpclientfactory]: https://learn.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests "Using HttpClientFactory to add resiliency in Apps"
[polly]: https://github.com/App-vNext/Polly "Polly resiliency and fault handling library"
[mer]: https://www.nuget.org/packages/Microsoft.Extensions.Resilience "NuGet.org Microsoft.Extensions.Resilience"
[mehr]: https://www.nuget.org/packages/Microsoft.Extensions.Http.Resilience  "NuGet.org Microsoft.Extensions.Http.Resilience"
[defaults]: https://github.com/dotnet/extensions/blob/main/src/Libraries/Microsoft.Extensions.Http.Resilience/README.md#usage-examples "Defaults for Standard ResilienceHandler"
