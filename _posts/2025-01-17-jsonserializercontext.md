---
layout: post
title: Working with JsonSerializerContext in System.Text.Json and Refit
author: Tomasz Cielecki
comments: true
date: 2025-01-17 9:30:00 +0100
tags:
- dotnet
- sourcegen
- refit
---

Recently I have added `<EnableTrimAnalyzer>true</EnableTrimAnalyzer>` to a bunch of my projects, which now yield warnings when code I wrote has potential issues when trimming the assembly.

What is trimming? In short terms, instead of shipping all the code in all assemblies in an Release version of your App, the trimming process scans what is actually used and cuts out all the other code that is unused.
This leads to much smaller Apps in terms of size. There are other processes which also help with speed.

Additionally, in a mobile context, we often opt into AOT compilation, which also brings a set of gotchas. For example, if your App uses reflection or emits code, it will likely not work at runtime. To replace a lot of this kind of code, some time ago we got Source Generators to help us do some of these things at compile time, instead of doing them at runtime.

So because of enabling the Trim Analyzers, I now noticed a few warnings around serialization code I had, which told me I could use `JsonSerializerContext` to source generate the serializers to help with trimming but also speed. Whenever I read that something becomes faster, I get excited. I love speed! So, obviously I had to try this out.

So commonly when serializing/deserializing something with System.Text.Json you could have some code looking something like this:

```csharp
record PersonDto(string Id, string Name);

var person = JsonSerializer.Deserialize<PersonDto>(json);
```

This code would now emit a Trimmer warning because the definition of the method looks as follows.

[![][api-def]][api-def]

If you look carefully, there are two annotations that trigger the trimming warnings. Namely `[RequiresUnreferencedCode]` and `[RequiresDynamicCode]`. These essentially say, that the method uses code outside of its own knowledge and that this code could potentially be trimmed away and code will be generated at runtime, which is not compatible with (Native)AOT.

OK, what then? Source generators to the rescue! So to avoid the generated code at runtime and to avoid types getting trimmed, we can implement `JsonSerializerContext` and tell the serializer about this. Which is fairly simple.

```csharp
[JsonSerializable(typeof(PersonDto))]
internal sealed partial SerializerContext : JsonSerializerContext;
```

This will generate the serializer needed to serialize and deserialize `PersonDto`. So this means, that any type you want to run through the serializer, you would need to add to a `JsonSerializerContext`. Additionally,
you need to use the method overload that takes the `JsonSerializerContext` when serializing or deserializing. So the example from above becomes:

```csharp
PersonDto person = JsonSerializer.Deserialize(json, SerializerContext.Default.PersonDto);

// or

PersonDto person = JsonSerializer.Deserialize(bayJson, typeof(PersonDto), SerializerContext.Default);
```

Obviously there are async variants as well, however, they will have a very similar signature to do the same.

So with these few steps you are now trimmer safe and AOT compatible when serializing and deserializing code with System.Text.Json.

## Usage in Refit

If you are a user of Refit. The default serializer used is System.Text.Json, with its default configuration. If you want to use your newly added `JsonSerializerContext` you need to tell Refit about his using its settings. This can be done simply with something like this:

```csharp
var settings = new RefitSettings
{
    ContentSerializer = new SystemTextJsonContentSerializer(SerializerContext.Default.Options)
};

var api = RestService.For<IPersonApi>(url, settings);
```

Now you are good to go and Refit will use your context. However, there are a few caveats to be aware of.

On the `JsonSerializerContext` you can set some options through `JsonSourceGenerationOptions`, like how to format json when serializing, or whether to allow case insensitivity on fields. So applied on the `JsonSerializerContext` from before, this could look like:

```csharp
[JsonSourceGenerationOptions(PropertyNameCaseInsensitive = true, WriteIndented = true)]
[JsonSerializable(typeof(PersonDto))]
internal sealed partial SerializerContext : JsonSerializerContext;
```

These options are however ignored if you accidentally create your settings like this:

```csharp
var settings = new RefitSettings
{
    ContentSerializer = new SystemTextJsonContentSerializer(
        new JsonSerializerOptions
        {
            TypeInfoResolver = SerializerContext.Default
        })
};

var api = RestService.For<IPersonApi>(url, settings);
```

So make sure to provide the full options like shown in the first example with `RefitSettings`.

If you are using `HttpClientFactory` and a `ServiceCollection` to register your Refit APIs you can also pass settings with:

```csharp
serviceCollection.AddRefitClient<IPersonApi>(settings);
```

[api-def]: {{ site.url }}/assets/images/trimming/api-def.png "Screenshot from Rider showing the definition of the method, and showing the method has been annotated with RequiresUnreferencedCode and RequiresDynamicCode"
