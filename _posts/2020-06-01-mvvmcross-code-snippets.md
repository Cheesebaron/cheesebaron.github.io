---
layout: post
title: MvvmCross Code Snippets
author: Tomasz Cielecki
comments: true
date: 2020-06-01 00:00:00 +0000
tags:
- Xamarin
- MvvmCross
---

This blog post is a part of Louis Matos's Xamarin Month, where this months topic is Code Snippets. For more information [take a look at his blog][lmts] and see the list of all the other auhtors who are participating. There will be a new post each day of the month, which is super cool!

Let me share some code snippets that I often use. All of these snippets will be available in my [XamarinSnippets code repository on GitHub][snippets], for import in Visual Studio 2019, ReSharper, Rider and Visual Studio for Mac. Instructions provided in the repository Readme file.

When writing an Application using MvvmCross, or even with other frameworks, there are some pieces of code that you have to repeat again and again. As programmers we are usually a bit lazy and don't want to type all that code over and over again. Fortunately, we can have code snippets ready to help us, a nice feature built into our IDE's. Some of the snippets I use often are as follows.

# mvxprop

I often have to create a property in my ViewModels which raise the `PropertyChanged` event in the MvvmCross flavor. For that I simply type **mvxprop** and press Tab, and magically I get the following code.

```csharp
private int propertyName;
public int PropertyName
{
    get => propertyName;
    set => SetProperty(ref propertyName, value);
}
```

I have considered making some flavors for some common types that I use, such as string, bool, int. Not sure how much I would use them.

# mvxcom

Creating commands is also something that I often do. However, I've gone away from using this pattern where I lazily initialize commands, some of you may find them useful. What I prefer instead is initialize them in the constructor of the ViewModel instead.

```csharp
private MvxCommand _command;
public MvxCommand Command =>
    _command ??= new MvxCommand(DoCommand);

private void DoCommand()
{
    // do stuff
}
```

I have a couple of variants of this snippet, which creates different types of `MvxCommand`. **mvxcomt** for the generic `MvxCommand<T>`, **mvxcomasync** for the async version `MvxAsyncCommand` and similarly the generic version of that **mvxcomtasync** which creates a `MvxAsyncCommand<T>`.

# mvxbset

Last but not least, a snippet for creating a binding set for binding views on both Android and iOS. I've taken the liberty to make the assumption that View and ViewModel names follow each other. So `PeopleView` will usually have a corresponding ViewModel `PeopleViewModel`. This ends up creating something like this:

```csharp
using var set = this.CreateBindingSet<PeopleView, PeopleViewModel>();
```

Get all the snippets along with instructions in my [XamarinSnippets repository on GitHub][snippets].

Do you have any cool related MvvmCross snippets to share? Put them in the comments or make a Pull Request on the repository.

[lmts]: https://luismts.com/code-snippetss-xamarin-month/# "Louis Matos - Xamarin Month - Code Snippets"
[snippets]: https://github.com/Cheesebaron/XamarinSnippets "XamarinSnippets code repository on GitHub"