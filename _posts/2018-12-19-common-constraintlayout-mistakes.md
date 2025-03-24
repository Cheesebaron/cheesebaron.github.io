---
layout: post
title: Some Common ConstraintLayout Pitfalls and Mistakes
author: Tomasz Cielecki
comments: true
date: 2018-12-19 21:25:00 +0100
tags:
- Xamarin
- Xamarin.Android
- dotnet
---

Recently I have been working a lot with converting nested layouts, into flattened layouts using [ConstraintLayout][cl] as their root element.

If you do not know what `ConstraintLayout` is all about, in short terms, it is a layout that allows to position and size views in a flexible manner. It is in some ways similar to RelativeLayout. However, it gives you more power by allowing you to use percentages for sizes, chaining views together and much more.

Since I have been working with it quite a bit, I would try to highlight some common mistakes that people make when using ConstraintLayout. Also, help you avoid some pitfalls you can encounter on your. Hopefully, this will help you improve your own layouts.

## Do not use `match_parent` for widths and heights

When you use ConstraintLayout, you are constraining your view dimensions using rules. These rules are a family of attributes that you apply on your view, starting with `layout_constraint`. There are various constraints, such as:

```
layout_constraintTop_toTopOf
layout_constraintStart_toStartOf
```

A common mistake is to use `match_parent` for `layout_width` and `layout_height`. However, what you instead should use is `0dp` which means _*match contstraints*_, because `ConstraintLayout` the entire idea about this layout is that you apply constraints to size and position your views.

Instead of `match_parent` for width, you would instead use the following two rules:

```
app:layout_constraintStart_toStartOf="parent"
app:layout_constraintEnd_toEndOf="parent"
```

Where `parent` is the encapsulating view, in this case it would be the ConstraintLayout. However, instead of writing `parent` it could be the ID of another view in the layout.

Now, `match_parent` will work and do what it does in other layouts. However, on some older versions of `ConstraintsLayout` it is not recommended to use it, since it is not entirely supported. Make sure you are using version 1.1 or newer.

## What about `wrap_content`?

`wrap_content` is still valid for `layout_width` and `layout_height` and will be respected by ConstraintLayout.

A thing to note though. If you use `wrap_content` and for instance the two rules, from the `match_parent` example above like:

```xml
<View
    android:layout_width="wrap_content"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent"
    ...
```

The view will not match the entire width of the `parent`. What you instead are describing here is that the view wraps its width, but should be positioned between the start and end of the `parent`.
This is useful with couple of other attribute called `layout_constraintHorizontal_bias` and `layout_constraintVertical_bias`.

Let us say you want to postion a view at a third of the screens width. You would apply the following constraints like so:

```xml
<View
    android:layout_width="wrap_content"
    app:layout_constraintHorizontal_bias="0.3"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent"
    ...
```
Pretty neat!

## Broken chains

ConstraintLayout allows you to create a chain between multiple views. This can help you with things like:

- Equally spreading chained views

```
+                                                      +
|    +----------+     +----------+     +----------+    |
|    |          |     |          |     |          |    |
|    |          +----->          +----->          |    |
<----+  View A  |     |  View B  |     |  View C  +---->
|    |          <-----+          <-----+          |    |
|    |          |     |          |     |          |    |
|    +----------+     +----------+     +----------+    |
+                                                      +
```

For this effect you need to apply `layout_constraintHorizontal_chainStyle="spread"` or `layout_constraintHorizontal_chainStyle="spread_inside"` on `View A`. The latter will not add spacing at the start of `View A` and at the end of `View C`. There is also a `Vertical` variant of this chain style.


- Weigh views similarly to what `LinearLayout` can do

```
+                                   +
|+----------++---------------------+|
||          ||                     ||
||  Weight  +>        Weight       ||
<+   0.25   ||         0.75        +>
||          <+                     ||
||          ||                     ||
|+----------++---------------------+|
+                                   +
```

Apply `layout_constraintHorizontal_weight` or `layout_constraintVertical_weight` on all elements in the chain to add weights.

- Packing together views

```
+                                                    +
|        +----------++----------++----------+        |
|        |          ||          ||          |        |
|        |          +>          +>          |        |
<--------+  View A  ||  View B  ||  View C  +-------->
|        |          <+          <+          |        |
|        |          ||          ||          |        |
|        +----------++----------++----------+        |
+                                                    +
```

Some of the variants can have bias applied to them to increase or decrease the effects on elements. Say you want to spread equally, except for the center view, which needs more space.

If you notice the small ascii drawings above, in between all of the views there is a directional arrow. In order to create a chain, you need to create constraints that position your views bi-directional. So if you have views `A`, `B` and `C`. `A` _must_ have a constraint to a `parent` or another view _and_ to `B`. In turn `B` _must_ a constraint to `A` and to `C`. At last `C` must have a constraint to `B` and to another view or `parent`. If you forget to constraint bi-directinally, your chain is broken and the effect you want to achieve will not work.

So compared to `RelativeLayout` which does not allow to add circular positioning between views `ConstraintLayout` actually allows this in order to create these chains.

## Setting Guideline percentages based on screen size and orientation

`ConstraintLayout` has a couple of virtual layouts, which are not visible on the screen at all. `Guideline` being one of the. `Guideline`s are useful for positioning `start`, `end`, `top` and `bottom` constraints on a view against it. 

A guideline can be set at a fixed position using `layout_constraintGuide_begin="100dp"` which will position it at `100dp` at the left or top of a view. Or you can alternatively use `layout_constraintGuide_end="100dp"`, which will postion the Guideline at `100dp` to the right or bottom of a view.

A guideline can also be positioned at a percentage of the width or height of the view with `layout_constraintGuide_percent="0..1"` where `0` is left, `0.5` is middle and `1` is right.

One cool thing is that you can use Android's values folder structure to provide dimensions for different screen sizes for these guidelines.
For instance on a phone you want to have a view fill most of the screen, while on a table you may want to to only fill half of the screen.

This is easy to set up for dimensions specified in `dp`. However, for the percentages, it is slightly different. Plain `dimens` on Android do not allow you to use floating point values.

Imagine you have a folder structure as follows:

```
Resources
    values
        dimens.xml
    values-sw600dp
        dimens.xml
```

You would put your normal phone resources into `values/dimens.xml` while you would put your tablet resources into `values-sw600dp/dimens.xml`.

Regular dimens for typed values like `sp`, `dp`, `px` can be added to a `dimens.xml` file like so:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
  <dimen name="guidelineBegin">16dp</dimen>
</resources>
```

This can be consumed for your `Guideline` like so:

```xml
<android.support.constraint.Guideline
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    app:layout_constraintGuide_begin="@dimen/guidelineBegin" />
```

The dimension added in `values/dimens.xml` will be substituted by the value in `values-sw600dp/dimens.xml` if the `sw`, smallest width is `600dp` or more. There are other values folders you can use, which you can read more about in the [Android App resources overview][resources].

Dimensions which are float values, normally need to be followed by a typed value `sp`, `dp`, etc. In order to add percentages we can use for `Guideline`s, these need to be specified slightly differently like so:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
  <item name="fiftyPercent" type="dimen">0.5</item>
</resources>
```

These can still be consumed as normal dimensions like so:

```xml
<android.support.constraint.Guideline
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    app:layout_constraintGuide_percent="@dimen/fiftyPercent" />
```

## Should you use `ConstraintLayout` for all your layouts?

`ConstraintLayout` is very powerful and allows you to create complex layout, without a nested hell of views. Often it can be much more performant than a deeply nested layout. However, should you use it even for the simplest layouts? Obviously it really depends on the use case. However, it should be noted that all this power does not come for free. There is some overhead in calculating positions and sizes for views in your layout when using `ConstraintLayout`. Hence, in many cases using a flattened layout with `RelativeLayout` as root, may outperform `ConstraintLayout`.
So, in the end it is a question of the complexity of the layout and how much of the power `ConstraintLayout` provides you want to utilize.

There are various benchmarks out there. Make sure the benchmark you read include both inflation-, layout- and measure-time. Also you may find that performance differs significantly between versions benchmarked as improvements find their way into `ConstraintLayout` over time.
You can read more about the performance benefits on the [Android Developers Blog][apb].

Hopefully you found some of this information useful.

[cl]: https://developer.android.com/reference/android/support/constraint/ConstraintLayout "ConstraintLayout Reference"
[resources]: https://developer.android.com/guide/topics/resources/providing-resources "Android Developers App resources overview"
[apb]: https://android-developers.googleblog.com/2017/08/understanding-performance-benefits-of.html "Android Developers Blog, `ConstraintLayout` performance benefits"
