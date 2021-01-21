---
layout: post
title: "UIView styling with functions"
author: "Pierre Felgines"
---

Today I want to talk about `UIView` styling. The common approach when we want to customize the display of native UI controls (for instance `UIButton` or `UILabel`) is to create a subclass that overrides a bunch of properties. That works well most of the time, but some problems may arise.

## The filled and rounded button

We you use subclasses in Swift to style your views, you loose composition. For example, if you create two `UIButton` subclasses, `FilledButton` and `RoundedButton`, how do you create a button that is both filled and rounded ?

One solution of this problem is to stop using subclasses, but to leverage Swift functions and type system.

We can think about a view style, as a function that consumes a view, and set some properties on it.

{% highlight swift %}
let filledButtonStyle: (UIButton) -> Void = {
    $0.setTitleColor(.white, for: .normal)
    $0.backgroundColor = .red
}

let button = UIButton()
button.setTitle("My Button", for: .normal)
filledButtonStyle(button)
{% endhighlight %}

{% include image.html
            img="assets/style_filled_button.png"
            title="Image 1. Filled button"
            caption="Image 1. Filled button" %}

We can wrap this function into an object, for more control on it.

{% highlight swift %}
struct ViewStyle<T> {
    let style: (T) -> Void
}
{% endhighlight %}

We can now create some styles for our filled and rounded button.

{% highlight swift %}
let filled = ViewStyle<UIButton> {
    $0.setTitleColor(.white, for: .normal)
    $0.backgroundColor = .red
}

let rounded = ViewStyle<UIButton> {
    $0.layer.cornerRadius = 4.0
}

let button = UIButton()
filled.style(button)
{% endhighlight %}

Now that we have our two styles for both filled and rounded buttons, we can create a new style for a rounded and filled button very easily.

{% highlight swift %}
extension ViewStyle {

    func compose(with style: ViewStyle<T>) -> ViewStyle<T> {
        return ViewStyle<T> {
            self.style($0)
            style.style($0)
        }
    }
}

let roundedAndFilled = filled.compose(with: rounded)
{% endhighlight %}

What was previously impossible with `UIButton` subclasses is now very straightforward using simple functions.

{% include image.html
            img="assets/style_rounded_button.png"
            title="Image 2. Filled and rounded button"
            caption="Image 2. Filled and rounded button" %}

## Improvements

Now that we get the general idea, it's time for syntactic sugar!

First of all, for now our styles live in the global namespace. That's not very scalable.

The solution here is to extend `ViewStyle` and to constrain the generic type.

{% highlight swift %}
extension ViewStyle where T: UIButton {

    static var filled: ViewStyle<UIButton> {
        return ViewStyle<UIButton> {
            $0.setTitleColor(.white, for: .normal)
            $0.backgroundColor = .red
        }
    }

    static var rounded: ViewStyle<UIButton> {
        return ViewStyle<UIButton> {
            $0.layer.cornerRadius = 4.0
        }
    }

    static var roundedAndFilled: ViewStyle<UIButton> {
        return filled.compose(with: rounded)
    }
}
{% endhighlight %}

That's nice, we have a namespace to list all our styles. But it's not very handy to style a button yet.

{% highlight swift %}
ViewStyle<UIButton>.roundedAndFilled.style(button) // 🙈
{% endhighlight %}

To improve this, we can define a function that is responsible to apply a style to an object, inferring the type of the style based on the type of the object.

{% highlight swift %}
func style<T>(_ object: T, with style: ViewStyle<T>) {
    style.style(object)
}

style(button, with: .roundedAndFilled)
{% endhighlight %}

## Protocols to the rescue

The code looks good and is readable. But we can go one step further! I want to get rid of the global `style(_:with:)` function, and to use an instance method of `UIButton` instead. For this, let's define an empty protocol `Stylable`, and make `UIView` conform to it. That way we will be able to add methods to `Stylable` and all the `UIView` subclasses will get them for free.

{% highlight swift %}
protocol Stylable {}

extension UIView: Stylable {}
{% endhighlight %}

That may seem a little odd, but we can now extend `Stylable` to add a method to apply a style to any `Stylable` instance.

{% highlight swift %}
extension Stylable {

    func apply(_ style: ViewStyle<Self>) {
        style.style(self)
    }
}
{% endhighlight %}

All the `UIView` subclasses gain this `apply(_:)` method for free! The code becomes compact and readable.

{% highlight swift %}
button.apply(.roundedAndFilled)
{% endhighlight %}

What's more, we can't misuse our styles because of the Swift type system!

{% highlight swift %}
let labelStyle = ViewStyle<UILabel> { $0.textAlignment = .center }
button.apply(labelStyle) // 💣

// error: cannot convert value of type 'ViewStyle<UILabel>' to expected argument type 'ViewStyle<UIButton>'
{% endhighlight %}

## Init with style

With the previous `apply(_:)` method you will often find yourself writing these two lines:

{% highlight swift %}
let button = UIButton()
button.apply(.rounded)
{% endhighlight %}

What if we could initialize our button (or any other `UIView`) with a predefined style? It would save us one line of code each time.

That is possible, modifying slightly our `Stylable` protocol.

{% highlight swift %}
protocol Stylable {
    init()
}

extension UIView: Stylable {}

extension Stylable {

    init(style: ViewStyle<Self>) {
        self.init()
        apply(style)
    }

    func apply(_ style: ViewStyle<Self>) {
        style.style(self)
    }
}
{% endhighlight %}

We can now use the following syntax:

{% highlight swift %}
let button = UIButton(style: .roundedAndFilled) // 👌
{% endhighlight %}

## Conclusion

With view styles as plain Swift functions, we achieved two things:
- first, a technical improvement on view subclasses: the composition of two `UIView` subclasses was impossible, whereas it becomes very easy using Swift functions.
- second, an easier communication between developers and designers. Indeed, designers often work with styles, in order to reuse components and keep a consistent look and feel all around the app. If you can extract their styles and map them in a Swift file, it will become much simpler to develop your UI, and to update these styles in the future.

You can find the full gist [here](https://gist.github.com/felginep/0148b40e26b19d07e81c2e1e4f2ff3d2).


