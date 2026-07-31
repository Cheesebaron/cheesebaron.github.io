---
layout: post
title: Source Generated Refit Clients!
author: Tomasz Cielecki
comments: true
date: 2026-07-31 10:35:00 +0200
tags:
- Android
- iOS
- dotnet
- MAUI
---

I have previously blogged about [Refit and how to set up reslience][post] and it is no secret that I am a big fan of writing my client code using Refit.

For a long time though, Refit under the hood has been using reflection for a lot of its code. Even though a lot of your Refit client would be source generated, there would still be a lot of reflection going on at runtime to build query string parameters and when serializing and deserializing your types.

This changed recently in [this massive Pull Request][pr], where a lot of the code to generate a client, instead of using reflection now inlines the logic for the queries and much more. To catch up on all the changes make sure to [have a look at the breaking changes documentation][breaking].

What this means in the end. Since Refit is now reflection-free, we can fully AOT compile all the code it emits and without trimming warnings leaking into your code, which you could not do much about before. This is huge! With the changes made for this release they have also spent a lot of effort on eliminating hot paths in the code. So in combination with AOT your Refit client should be blazing fast!

Main points if you adopt Refit v14, is to make a few minor changes to your code.

- Instead of using `RestService.For<T>`, use `RestService.ForGenerated<T>`
- If you are using HttpClientFactory instead of using `AddRefitClient` use `AddRefitGeneratedClient`
- If you are using models to provide query string parameters, Refit now also supports `[JsonPropertyName]` on that model instead of using `[AliasAs]`. Meaning you could have a model like so and the name of the paramerter would be picked up from the [`JsonPropertyName`]:

```csharp
record Filter([property: JsonPropertyName("file_name")] string FileName);

[Get("/search")]
Task<string> Search([Query] Filter filter);
```

[post]: {% post_url 2024-06-25-refit-resilience %}
[pr]: https://github.com/reactiveui/refit/pull/2210
[breaking]: https://github.com/reactiveui/refit/blob/main/docs/breaking-changes.md#v14xx