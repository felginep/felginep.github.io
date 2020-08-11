---
layout: post
title: "Make impossible states impossible"
author: "Pierre Felgines"
---

Swift is a strongly typed language and in this article I want to focus on how to strengthen some part of our code using Swift types.

# Optionals

Optionals are a powerful tool, but they quickly become a pain in one particular case: when the value of the object is set in a distant future and most of the logic relies on this value.

For example, let's imagine we are working on an application that fetches some articles. For an article we can share it or open it in a webview.

The code might look like this:

{% highlight swift %}
class ArticleViewController: UIViewController {

    private var article: Article?

    override func viewDidLoad() {
        super.viewDidLoad()

        fetcher.fetchArticle() { [weak self] result in
            switch result {
            case let .success(article):
                self.article = article
                // display article somehow
            case let .error(error):
                break // Handle error
            }
        }
    }

    // ...

    func shareButtonTapped() {
        guard let article = article else {
            preconditionFailure("Article should exists")
        }
        // Share article
    }

    func openInWebView() {
        guard let article = article else {
            preconditionFailure("Article should exists")
        }
        // Open article
    }
}
{% endhighlight %}

The `article` instance variable has to be declared as `Optional<Article>` for two reasons:
- the article will be available later in the lifecycle of the view controller, when the fetcher gets its result
- the fetcher may return an error instead of an article

What is annoying is that the *happy path* is when the article is available and stored in the `article` property. Most of the code will handle this case (the *unhappy path* is often a smaller part of the code).

The issue in the snippet above is the repeated `guard` statements for each action that wants to access the article.

Logically these actions can *only* exist when the article is not nil.

One solution in this case is to extract all the code that depends on the article into its own class. This new class represents the *happy path* and must be initialized with a non optional article.

{% highlight swift %}
class ArticleActionHandler {

    private let article: Article

    init(article: Article) {
        self.article = article
    }

    func share() {
        // Share article
    }

    func openInWebView() {
        // Open article
    }
}
{% endhighlight %}

This class is really simple and can be tested easily. Contrary to view controllers, all the dependencies the class need are passed in the `init` function.

The caller site becomes simpler too:

{% highlight swift %}
class ArticleViewController: UIViewController {

    private var actionHandler: ArticleActionHandler?

    override func viewDidLoad() {
        super.viewDidLoad()

        fetcher.fetchArticle() { [weak self] result in
            switch result {
            case let .success(article):
                self?.actionHandler = ArticleActionHandler(article: article)
                // display article somehow
            case let .error(error):
                break // Handle error
            }
        }
    }

    // ...

    func shareButtonTapped() {
        actionHandler?.share()
    }

    func openInWebView() {
        actionHandler?.openInWebView()
    }
}
{% endhighlight %}

Note that this technique is not reserved for view controllers, it can be applied everywhere. When you see that `preconditionFailure` start to be scattered all over the place, there is a need for refactoring.

# NonEmpty

Let's say we are in an app that has some sort of `Project` entity. A project contains some *products*, and a project cannot live without products (it has at least one product).

We can represent this relationship with the following structures:

{% highlight swift %}
struct Product {
    var name: String = ""

    static let empty = Product()
}

class Project {
    private(set) var products: [Product]

    init(products: [Product] = [.empty]) {
        self.products = products
    }

    func add(_ product: Product) {
        products.append(product)
    }
}
{% endhighlight %}

In the app there is a class responsible to edit a product. For example we can change the product name in the editor.

{% highlight swift %}
class ProductEditor {

    private let project: Project
    private var currentProduct: Product?

    init(project: Project) {
        self.project = project
        self.currentProduct = project.products.first
    }

    // ...

    func updateProductName() { ... }
}
{% endhighlight %}

The issue here is that our editor has an optional `currentProduct`. Based on the specifications of the `Project`class, a product should always exist in a project. `currentProduct` *can't* be nil. But there is no way to know it at compile time, that's why `currentProduct` is optional.

A possible solution would be to declare `var currentProduct: Product` and raise an error if the project does not contains any product:

{% highlight swift %}
class ProductEditor {

    private let project: Project
    private var currentProduct: Product

    init(project: Project) {
        self.project = project
        guard let product = project.products.first else {
            preconditionFailure("A project should contains at least one product")
        }
        self.currentProduct = product
    }

    // ...
}
{% endhighlight %}

But that does not feel right. The editor should not *know* that a project has at least one product, it is not its responsability.

One solution here is to create a type that represents the fact that a project must have at least one product.

We can create a `NonEmpty` type that does just this. The folks at Pointfree have covered the [topic](https://www.pointfree.co/episodes/ep20-nonempty) and even released a [library](https://github.com/pointfreeco/swift-nonempty) for this use case.

The core implementation of `NonEmpty` is really simple: a non empty list is composed of a first element (the `head`) and a possible empty list (the `tail`).

{% highlight swift %}
struct NonEmpty<C: Collection>: Collection {
    typealias Element = C.Element

    private var head: Element
    private var tail: C

    init(_ head: Element, _ tail: C) {
        self.head = head
        self.tail = tail
    }
}
{% endhighlight %}

What's beautiful about it, is that we cannot instantiate an empty collection. The `head` is always a non optional element.

With this structure, we can refactor our code to be more expressive and safer:

{% highlight swift %}
class Project {
    private(set) var products: NonEmpty<[Product]>

    init(products: NonEmpty<[Product]> = NonEmpty<[Product]>(.empty, [])) {
        self.products = products
    }

    func add(_ product: Product) {
        products.append(product)
    }
}
{% endhighlight %}

When someone reads the `Project` public API, he can instantly understand that the products will not be empty. It is written in the type, no more need for a comment.

The `NonEmpty` type can define a `safeFirst` accessor to access the head as a non optional element:

{% highlight swift %}
extension NonEmpty {
    var safeFirst: Element {
        return self.head
    }
}
{% endhighlight %}

{% highlight swift %}
class ProductEditor {

    private let project: Project
    private var currentProduct: Product

    init(project: Project) {
        self.project = project
        self.currentProduct = project.products.safeFirst
    }

    // ...
}
{% endhighlight %}

Note that this time the `currentProduct` instance is still not an optional but we removed the `preconditionFailure` too. The editor does not care about the relationships between a project and the list of products, it is now constrained by the available API.

Creating your own types is really powerful and is often another way to remove `preconditionFailure` in your code. But sometimes it is not always possible to define such types.

# Enums

Let's face it, code is rarely bad from the start. The issue is that it grows incrementally and is often changed by multiple developers.

That's why we may find ourselves having difficulty keeping track of the current state in an application.

For instance imagine a simplified situation like this:

{% highlight swift %}
class ArticleViewController: UIViewController {

    private var isLoading: Bool = true
    private var result: Result<Article, Error>?

    // ...
}
{% endhighlight %}

If we think about it, when you load an article, the `result` variable should not exist. And once `result` is set, the `isLoading` should always be false.

For now, 6 states are possible:
- `isLoading == true` and `result == nil`
- `isLoading == true` and `result == .success`
- `isLoading == true` and `result == .failure`
- `isLoading == false` and `result == nil`
- `isLoading == false` and `result == .success`
- `isLoading == false` and `result == .failure`

Adding a boolean state in the instance variables should be an alert to think if it's time for refactoring. In this case we should consider lowering the possible states with an enum:

{% highlight swift %}
enum ContentState<T> {
    case loading
    case loaded(Result<T, Error>)
}
{% endhighlight %}

The view controller state becomes simpler:

{% highlight swift %}
class ArticleViewController: UIViewController {

    private var state: ContentState<Article> = .loading

    // ...
}
{% endhighlight %}

With this change, only 3 states are possible:
- `loading`
- `loaded` with `success`
- `loaded` with `failure`

We removed 3 states from the possible outputs. It is now impossible to be loading while a result exists and vice versa.

# Pushing responsibility away

When designing an API you should also consider what types you expect. Let's say you have an application that takes the initials of an user, and creates some fancy images with the initials (for instance a placeholder profile picture for the user). For simplicity, let's assume the initials are always composed of two letters.

The first idea that comes in mind is to represent initials with a string like this:

{% highlight swift %}
class InitialsViewController: UIViewController {

    var initials: String?

    func initialsTextFieldChanged(_ text: String) {
        if text.count == 2 {
            initials = text
        } else {
            initials = nil
        }
    }
}
{% endhighlight %}

For now this seems fine, we only create initials if there are two characters in the string.

Later in the application we want to generate a profile picture out of the initials.

{% highlight swift %}
func generateProfilePicture(initials: String) -> UIImage {
      guard initials.count == 2,
        let firstLetter = initials.first,
        let secondLetter = initials.last else {
            preconditionFailure("Initials must have two letters")
      }
      return image(from: firstLetter).compose(with: image(from: secondLetter))
}
{% endhighlight %}

Note the `preconditionFailure`. In this case initials are supposed to be already valid, meaning having only two characters. But the input type `String` is too wide and allows the caller of the function to pass invalid data. That's why we need to re-validate the content before using it.

To avoid this duplication, let's consider creating a type for our initials and pushing the responsibility of creating and validating initials outside of the `generateProfilePicture` function.

{% highlight swift %}
struct Initials {
    let firstLetter: Character
    let secondLetter: Character
}
{% endhighlight %}

The `generateProfilePicture` is now very simple and do not care about the generation of initials:

{% highlight swift %}
func generateProfilePicture(initials: Initials) -> UIImage {
    return image(from: initials.firstLetter)
        .compose(with: image(from: initials.secondLetter))
}
{% endhighlight %}

There is no need to validate the initials because the type itself does not allow any bad inputs.

All the responsibility of creating initials is now at a single place:

{% highlight swift %}
extension Initials {
    init?(_ content: String) {
        guard content.count == 2,
            let firstLetter = content.first,
            let secondLetter = content.last else {
                return nil
        }
        self.firstLetter = firstLetter
        self.secondLetter = secondLetter
    }
}

class InitialsViewController: UIViewController {

    var initials: Initials?

    func initialsTextFieldChanged(_ text: String) {
        initials = Initials(text)
    }
}
{% endhighlight %}

# Conclusion

Many times, we use an assertion (like `preconditionFailure` or `assert`) to stop the program execution if a path should not happen. But assertions are not the best tool because they are very fragile. Indeed they raise a runtime error instead of a compilation error. It's very easy to duplicate code and to introduce bugs.

To avoid misusing assertions, we saw in this article different solutions:
- create a type that match your requirements and enforce the constraints (`NonEmpty`, `Initials`)
- create a enum to limit the number of states
- extract happy path code into its own class

