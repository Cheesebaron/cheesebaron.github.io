---
layout: post
title: MvvmCross Binding Target
author: Tomasz Cielecki
comments: true
date: 2018-1-15 15:00:00 +0100
tags:
- Xamarin
- MvvmCross
- Binding
- dotnet
---

I had some colleagues, who where a bit confused about what target is when creating bindings. Lets first establish what Source and Target means when talking about a binding:

- `Target`, the Property on your View you are binding
- `Source`, the Property in your ViewModel your View binds to

So `Target` will be any public property on your View, that you want to bind. Examples of these are `Text`, `ItemsSource`, `SelectedItem` and many more. Keep in mind that any public property when using MvvmCross on a View can be bound in `OneWay` mode.

To confuse things a bit, MvvmCross uses something we call `TargetBinding`, to create a definition of how a `View` can bind, when wanting modes other than `OneWay`. This means, if you want to bind something in `TwoWay` mode, there needs to exist a `TargetBinding` which describes how to achieve this.
Internally in a `TargetBinding`, what usually happens is that it simply subscribes `EventHandler`s to relevant events, in order to notify when something has changed on the `View` in order to feed them back to the `ViewModel`. You can [read more about how to create your own `TargetBinding` in the MvvmCross documentation about Custom Data Binding.](https://www.mvvmcross.com/documentation/fundamentals/custom-data-binding)

The `Source` in your `ViewModel`, will also need to be a public property. Similarly to every other MVVM framework, if you implement `INotifyPropertyChanged` on your `ViewModel` and remember to fire the `NotifyPropertyChanged` event when updating your properties, the MvvmCross binding engine will figure out how to feed the value to the `Target`.

## Binding Expressions

OK! Now, we have established `Target`, `Source` and `TargetBinding`. Now, let us look a bit into MvvmCross binding expressions.

### Swiss/Tibet (Android AXML, iOS string bindings etc.)

Android bindings and some string binding descriptions you can find for `MvxTableViewCell` and the likes use [Swiss/Tibet binding expressions](https://www.mvvmcross.com/documentation/fundamentals/data-binding), which is a description language specific to MvvmCross.

```xml
<SomeView
  local:MvxBind="ViewProperty ViewModelProperty" />
```

`SomeView` in this case is the bindable object, the View. `ViewProperty` is the `Target`, `ViewModelProperty` is the `Source`.

When binding `MvxTableViewCell`, the bindable object, will be the cell itself. Hence, the public properties you bind to need to be on the cell.

### XAML

In XAML binding expressions as per usual XAML bindings will look something as follows.

```xml
<SomeView ViewProperty="{Binding ViewModelProperty}" />
```

`SomeView` in this case is the bindable object, the View. `ViewProperty` is the `Target`, `ViewModelProperty` is the `Source`.

When using MvvmCross XAML BindingEx extensions, which uses Swiss/Tibet, where you can alternatively bind like so.

```xml
<SomeView mvx:Bi.nd="ViewProperty ViewModelProperty" />
```

### Fluent Bindings

MvvmCross also provides something called Fluent Bindings to create type strong binding expressions in code behind. For these to work, the place you create the fluent binding, needs to be a derivative of `IMvxBindingContextOwner`, which means the type will have a `BindingContext` property, which the Fluent Binding will use to find the `Source`.

These Fluent Bindings usually look something as follows.

```csharp
var set = this.CreateBindingSet<ContextOwnerType, ViewModelType>();
set.Bind(someView)
  .For(view => view.ViewProperty)
  .To(viewModel => viewModel.ViewModelProperty);
```

The `ContextOwnerType` is usually a `MvxViewController`, `MvxTableViewCell` or `MvxActivity`. This is typically the type containing the views you want to bind to.

When using the `BindingSet.Bind()` method, it expects the `Target` as argument, typically the actual `View` you want to bind. I have seen a couple of cases where someone tries to use the `ContextOwnerType` instead like so.

```csharp
public class MyTableViewCell : MvxTableViewCell
{
    ...
    private UILabel _someView;

    private void CreateBindings()
    {
        this.DelayBind(() => 
        {
            var set = this.CreateBindingSet<MyTableViewCell, SomeViewModel>();
            set.Bind(this)
                .To(cell => cell._someView.Text)
                .To(viewModel => viewModel.SomeProperty);
            set.Apply();
        });
    }
}
```

Now, you may know that in order for MvvmCross to be able to bind something it needs to be a public property. So the above binding expression will _not_ work, since `_someView` is not a public property. Another problem with that binding description is that `MvxPropertyInfoTargetBindingFactory` will attempt to create the binding using by trying to find the `ViewProperty` on the `Target`. This means if you use `set.Bind(ViewContainer).For(vc => vc.SomeView.Prop)` it will attempt to find `Prop` on `ViewContainer` and not `SomeView`, which `ViewContainer` contains.

So the conclusion is, when you use Fluent Bindings, what you use as `Target` needs to be the actual `View` you want to bind to or you will need to expose what you want to bind as a public property. So to fix the binding above you can do it by either:

1. Exposing the `Text` property on `_someView` as a public property and bind to that
2. Change the binding from using `this` to `_someView`

So for the first suggestion it will look like:

```csharp
public class MyTableViewCell : MvxTableViewCell
{
    ...
    private UILabel _someView;

    public string Text 
    { 
        get => _someView.Text;
        set => _someView.Text = value;
    }

    private void CreateBindings()
    {
        this.DelayBind(() => 
        {
            var set = this.CreateBindingSet<MyTableViewCell, SomeViewModel>();
            set.Bind(this)
                .To(cell => cell.Text)
                .To(viewModel => viewModel.SomeProperty);
            set.Apply();
        });
    }
}
```

For the second suggestion it will look like:

```csharp
public class MyTableViewCell : MvxTableViewCell
{
    ...
    private UILabel _someView;

    private void CreateBindings()
    {
        this.DelayBind(() => 
        {
            var set = this.CreateBindingSet<MyTableViewCell, SomeViewModel>();
            set.Bind(_someView)
                .To(view => view.Text)
                .To(viewModel => viewModel.SomeProperty);
            set.Apply();
        });
    }
}
```

One caveat with the first solution is. Since, the target is `MvxTableViewCell` we are using the default public property `One-Way` bindings. This means if `UILabel` was `UITextField` instead and we wanted to get changes when someone edits the `UITextField` in our `ViewModel`, this would not work, since there does not exist any `TargetBinding` classes, which describe this behavior for `MvxTableViewCell`. Hence, you will need to add your own for `MvxTableViewCell` or you will need to use the actual `UITextField` as `Target`.

In short terms, `Target` in a binding matters a lot!

Thank you for reading, if you want to read a bit more about bindings, value converters and value combiners in MvvmCross. Take a look at the official documentation about this topic.

- [Data binding](https://www.mvvmcross.com/documentation/fundamentals/data-binding)
- [Value converters](https://www.mvvmcross.com/documentation/fundamentals/value-converters)
- [Value combiners](https://www.mvvmcross.com/documentation/fundamentals/value-combiners)
- [Custom bindings (`TargetBinding`)](https://www.mvvmcross.com/documentation/fundamentals/custom-data-binding)
