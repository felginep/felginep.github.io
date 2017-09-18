---
layout: post
title:  "Registerable Views"
author: "Pierre Felgines"
---

This article was first published on Fabernovel [blog](https://en.fabernovel.com/insights/tech-en/registerable-views-2).

One of the most used UI component in iOS development is `UITableView`. This view is intended to present a collection of cells with a vertical layout. Its counterpart `UICollectionView` does exactly the same thing for more custom layouts (grid for example).

In the following, we will call _CollectionView_ an instance of `UITableView` or `UICollectionView`.

Every iOS developer knows that registering and dequeuing cells from a _CollectionView_ is kind of a boring task. A lot of boilerplate for a simple job: tell the CollectionView which cell we want to be able to use later.

As we start to embrace Swift in our daily work at Applidium, I would like to share with you a playground I’ve been playing with recently.

This is a swifty API to register and dequeue reusable cells (think `UITableViewCell` and `UICollectionViewCell`).

# Either a Class or a Nib

As we all know, a reusable cell is created either from a Nib or from a Class. We can represent this two cases with an enum.

{% highlight swift %}
enum RegisterableView {
    case nib(NSObject.Type)
    case `class`(NSObject.Type)
}
{% endhighlight %}

Our `RegisterableView` enum takes a class as associated value, to be able to provide it to the _CollectionView_ later.
Now we can add an extension to this enum with two computed variables: `nib` and `cellClass`.

{% highlight swift %}
extension RegisterableView {
    var nib: UINib? {
        switch self {
        case let .nib(cellClass):
            return UINib(nibName: String(describing: cellClass), bundle: nil)
        default:
            return nil
        }
    }

    var cellClass: AnyClass? {
        switch self {
        case let .class(cellClass):
            return cellClass
        default:
            return nil
        }
    }
}
{% endhighlight %}

Last thing we need for our `RegisterableView` is an identifier. Indeed we have to provide a string as identifier for each cell we register. Most of the time we use the name of the class as identifier.

First we define a protocol which returns a static identifier

{% highlight swift %}
protocol ClassIdentifiable {
    static func identifier() -> String
}
{% endhighlight %}

And then we extend `NSObject` to return the string representation of `self` as identifier.

{% highlight swift %}
extension NSObject: ClassIdentifiable {
    static func identifier() -> String {
        return String(describing: self)
    }
}
{% endhighlight %}

Now we can get an identifier for any `NSObject` subclass. For example: `UITableViewCell.identifier()` returns `"UITableViewCell"`.
Finally, our `RegisterableView` can return its identifier.

{% highlight swift %}
extension RegisterableView {
    var identifier: String {
        switch self {
        case let .nib(cellClass):
            return cellClass.identifier()
        case let .class(cellClass):
            return cellClass.identifier()
        }
    }
}
{% endhighlight %}

That’s all we need to do with `RegisterableView`. Let’s now see how to register them!

# A generic approach

The APIs to register cells from `UITableView` and `UICollectionView` are very close but not identical and we want a generic one that we can use for both.

To abstract the concrete UIKit classes, let’s create a `CollectionView` protocol that can register cells, headers or footers.

{% highlight swift %}
protocol CollectionView {
    func register(cell: RegisterableView)
    func register(header: RegisterableView)
    func register(footer: RegisterableView)
}
{% endhighlight %}

We can right away add handy methods to register multiple registerable views at once.

{% highlight swift %}
extension CollectionView {
    func register(cells: [RegisterableView]) {
        cells.forEach(register(cell:))
    }

    func register(headers: [RegisterableView]) {
        headers.forEach(register(header:))
    }

    func register(footers: [RegisterableView]) {
        footers.forEach(register(footer:))
    }
}
{% endhighlight %}

Now that we have the generic API, the only thing that we need to do, is to make `UITableView` and `UICollectionView` conforms to `CollectionView`. This is straightforward because most of the work is already done.

{% highlight swift %}
extension UITableView : CollectionView {
    func register(cell: RegisterableView) {
        switch cell {
        case .nib:
            register(cell.nib, forCellReuseIdentifier: cell.identifier)
        case .class:
            register(cell.cellClass, forCellReuseIdentifier: cell.identifier)
        }
    }

    func register(header: RegisterableView) {
        switch header {
        case .nib:
            register(header.nib, forHeaderFooterViewReuseIdentifier: header.identifier)
        case .class:
            register(header.cellClass, forHeaderFooterViewReuseIdentifier: header.identifier)
        }
    }

    func register(footer: RegisterableView) {
        register(header: footer)
    }
}
{% endhighlight %}

{% highlight swift %}
extension UICollectionView : CollectionView {
    func register(cell: RegisterableView) {
        switch cell {
        case .nib:
            register(cell.nib, forCellWithReuseIdentifier: cell.identifier)
        case .class:
            register(cell.cellClass, forCellWithReuseIdentifier: cell.identifier)
        }
    }

    func register(header: RegisterableView) {
        register(supplementaryView: header, kind: UICollectionElementKindSectionHeader)
    }

    func register(footer: RegisterableView) {
        register(supplementaryView: footer, kind: UICollectionElementKindSectionFooter)
    }

    func register(supplementaryView view: RegisterableView, kind: String) {
        switch view {
        case .nib:
            register(view.nib, forSupplementaryViewOfKind:kind , withReuseIdentifier: view.identifier)
        case .class:
            register(view.cellClass, forSupplementaryViewOfKind:kind , withReuseIdentifier: view.identifier)
        }
    }
}
{% endhighlight %}

# Results

With this new API, we are able to transform this ugly snippet:

{% highlight swift %}
tableView.register(MyTableViewCell.self, forCellReuseIdentifier: String(describing: MyTableViewCell.self))
tableView.register(UINib(nibName: String(describing: NibTableViewCell.self), bundle: nil), forCellReuseIdentifier: String(describing: NibTableViewCell.self))
{% endhighlight %}

to this clean and readable one:

{% highlight swift %}
tableView.register(cells: [
    .class(MyTableViewCell.self)
    .nib(NibTableViewCell.self)
])
{% endhighlight %}

A nice improvement based on an enum and a protocol!

# Some thoughts about the dequeue

The dequeue works as usual, except that we need to use the static identifier method from `ClassIdentifiable`.

{% highlight swift %}
let cell = tableView.dequeueReusableCellWithIdentifier(MyTableViewCell.identifier(), forIndexPath: indexPath) as! MyTableViewCell
{% endhighlight %}

We can even go one step further and clean up the dequeue a bit. The following method on `UITableView` allows us to dequeue a cell for an indexPath without even specifying the identifier.

{% highlight swift %}
func dequeueCell<U: ClassIdentifiable>(at indexPath: IndexPath) -> U {
    return dequeueReusableCell(withIdentifier: U.identifier(), for: indexPath) as! U
}
{% endhighlight %}

The dequeue now simply becomes:
{% highlight swift %}
let cell: MyTableViewCell = tableView.dequeueCell(at: indexPath)
{% endhighlight %}

or

{% highlight swift %}
let cell = tableView.dequeueCell(at: indexPath) as MyTableViewCell
{% endhighlight %}

The type of the cell, and thus the identifier is infered by the compiler. We do not need the identifier nor the cast at the end anymore.

# Conclusion

Swift has powerful features such as enums, protocol extensions and strong type system that we can leverage to create new APIs.

You can find the full playground of `RegisterableView` [here]({{ site.url }}/assets/registerable_views.playground.zip).

_2017-09-17 Updated for Swift 3 syntax_