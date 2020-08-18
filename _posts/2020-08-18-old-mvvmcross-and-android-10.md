---
layout: post
title: Old MvvmCross versions and Android 10 Play Store requirement
author: Tomasz Cielecki
comments: true
date: 2020-08-18 22:30:00 +0200
tags:
- Xamarin
- MvvmCross
---

I have gotten questions from multiple people, about versions of MvvmCross prior to version 6.4.1. What they can do about being [forced to target Android 10, API 29, or newer from November 2nd, when Google stops accepting updates to Apps targeting lower API levels](https://support.google.com/googleplay/android-developer/answer/113469#targetsdk).

MvvmCross 6.4.1 introduced some changes to MvxLayoutInflater that are needed in order for it to work with Android 10. However, these changes are for obvious reasons not part of previous releases.

Luckily MvvmCross is written in a way where you can replace parts of it through virtual methods and implementing certain interfaces. This can be used to retrofit old versions of MvvmCross with newer patched versions of, in this case, MvxLayoutInflater.

You simply have to follow these 3 simple steps!

1. Go to the corner
2. Curl up in a ball
3. Cry 😭

Sorry, wait, no. Thats not it! Upgrade MvvmCross! But, if you really cannot upgrade then try these steps.

1. Grab latest [MvxLayoutInflater](https://github.com/MvvmCross/MvvmCross/blob/develop/MvvmCross/Platforms/Android/Binding/Views/MvxLayoutInflater.cs)
2. Create your own implementation of `MvxContextWrapper` which uses 1.
3. Create your own MvxActivity derivatives that uses ContextWrapper from 2.
4. Replace all MvxActivities with your own

OK, I lied that was 4 steps. You might also need a 5th step, where you target Android 10 in your App, otherwise all this work will be useless anyways...

Lets go through all steps.

## 1. Get latest MvxLayoutInflater
Grab latest [MvxLayoutInflater](https://github.com/MvvmCross/MvvmCross/blob/develop/MvvmCross/Platforms/Android/Binding/Views/MvxLayoutInflater.cs), change namespace to something more suitable in your App. Rename the class `FixedLayoutInflater` or whatever you prefer.

## 2. Create your own implementation of `MvxContextWrapper`
We have to supply our own `MvxContextWrapper` class, the code in there is super simple. It just tells Android which LayoutInflater to use. We want to use our `FixedLayoutInflater` in there.

```csharp
using System;
using Android.Content;
using Android.Runtime;
using Android.Views;
using MvvmCross.Binding.BindingContext;
using Object = Java.Lang.Object;

namespace Awesome.App
{
    [Register("awesome.app.FixedContextWrapper")]
    public class FixedContextWrapper : ContextWrapper
    {
        private LayoutInflater _inflater;
        private readonly IMvxBindingContextOwner _bindingContextOwner;

        public static ContextWrapper Wrap(Context @base, IMvxBindingContextOwner bindingContextOwner)
        {
            return new FixedContextWrapper(@base, bindingContextOwner);
        }

        protected FixedContextWrapper(Context context, IMvxBindingContextOwner bindingContextOwner)
            : base(context)
        {
            if (bindingContextOwner == null)
                throw new InvalidOperationException("Wrapper can only be set on IMvxBindingContextOwner");

            _bindingContextOwner = bindingContextOwner;
        }

        public override Object GetSystemService(string name)
        {
            if (string.Equals(name, LayoutInflaterService, StringComparison.InvariantCulture))
            {
                return _inflater ??=
                    new FixedLayoutInflater(LayoutInflater.From(BaseContext), this, null, false);
            }

            return base.GetSystemService(name);
        }
    }
}
```

## 3. Create your own MvxActivity derivate
Next step is to create your own `MvxActivity` derivate. Why? Because you need to supply that ContextWrapper we just created. If you were to simply inherit from `MvxActivity` and override `OnAttachContext`, which is where we supply the ContextWrapper, then it would still use the original `MvxContextWrapper`, since the only way to supply it is by calling `base.OnAttachContext`.

OK. So yank whatever MvxActivity type you are re-implementing. Here is a non-exhaustive list of types we had.

| MvvmCross Version | Type |
|-------------------|------|
| 4.4.0 | [MvxActivity](https://github.com/MvvmCross/MvvmCross/blob/4.4.0/MvvmCross/Droid/Droid/Views/MvxActivity.cs) |
| 4.4.0 | [MvxAppCompatActivity](https://github.com/MvvmCross/MvvmCross-AndroidSupport/blob/4.4.0/MvvmCross.Droid.Support.V7.AppCompat/MvxAppCompatActivity.cs) |
| 5.7.0 | [MvxActivity](https://github.com/MvvmCross/MvvmCross/blob/5.7.0/MvvmCross/Droid/Droid/Views/MvxActivity.cs) |
| 5.7.0 | [MvxAppCompatActivity](https://github.com/MvvmCross/MvvmCross/blob/5.7.0/MvvmCross-AndroidSupport/MvvmCross.Droid.Support.V7.AppCompat/MvxAppCompatActivity.cs) |

The part to replace is the contents of `OnAttachContext` which should look something like:

```csharp
protected override void AttachBaseContext(Context @base)
{
    if (this is IMvxAndroidSplashScreenActivity)
    {
        // Do not attach our inflater to splash screens.
        base.AttachBaseContext(@base);
        return;
    }
    base.AttachBaseContext(FixedContextWrapper.Wrap(@base, this));
}
```

With that done, now you just have to replace every inheritance from `MvxActivity` in your App with this implementation, except for any splash screens, not needed there.

You may have to think a bit here and modify the code to your needs, but these are the simplest steps I could come up with.