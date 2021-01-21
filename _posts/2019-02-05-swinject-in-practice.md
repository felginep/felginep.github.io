---
layout: post
title: "Swinject in practice"
author: "Pierre Felgines"
---


I guess you already have heard of Dependency Injection. Dependency injection (DI) is a software design pattern that implements Inversion of Control for resolving dependencies.

On iOS, one of the popular frameworks you can use for Dependency injection is [Swinject](https://github.com/Swinject/Swinject).

Today we will quickly cover the basics of using Swinject in your app in order to focus on two edge cases you may have been confronted to: **custom object scopes** and **domain specific assemblies**.

Let's dive in.

## The basics

*Note*: I invite you to read the detailed [documentation](https://github.com/Swinject/Swinject/tree/master/Documentation) if you want to fully understand what follows.

Swinject let you split your dependencies into logic related objects, called *assemblies*.

For instance, let's create an `HelperAssembly` that registers few helpers dependencies, that we will need in our project.

{% highlight swift %}
class HelperAssembly: Assembly {

    func assemble(container: Container) {

        container.register(UIApplication.self) { _ in
            UIApplication.shared
        }

        container.register(UserDefaults.self) { _ in
            UserDefaults.standard
        }

        container.register(Bundle.self) { _ in
            Bundle.main
        }

        container.register(FileManager.self) { _ in
            FileManager.default
        }

        ...
    }
}
{% endhighlight %}


Swinject, like any DI framework, works like a key value store (named *container* here): the keys are types (abstract with protocols or concrete with classes or structs) and the values instances of those types.

Once low level dependencies are registered, we can start using them into higher level dependencies.

Let's say we need to fetch a list of the user last viewed products in our app. We create an interface (a protocol in Swift) and a concrete implementation of this interface.

{% highlight swift %}
// the interface
protocol LastViewedProductsRepository {
    ...
    func items() -> [Product]
}

// the implementation
class LastViewedProductsRepositoryImplementation: LastViewedProductsRepository {

    private let userDefaults: UserDefaults

    init(userDefaults: UserDefaults) {
        self.userDefaults = userDefaults
    }

    // MARK: - LastViewedProductsRepository

    func items() -> [Product] {
        // retrieve products from the userDefaults
    }
}
{% endhighlight %}

With Swinject, we would register our repository like this:

{% highlight swift %}
class RepositoryAssembly: Assembly {

    func assemble(container: Container) {

        // We register the abstract type as a key ...
        container.register(LastViewedProductsRepository.self) { r in
            // ... and a concrete implementation instance as a value
            LastViewedProductsRepositoryImplementation(
                // we don't know how to retrieve a userDefault,
                // we just expect Swinject to give us one at runtime
                userDefaults: r.resolve(UserDefaults.self)!
            )
        }.inObjectScope(.container)

        ...
    }
}
{% endhighlight %}

The `LastViewedProductsRepository` uses an instance of `UserDefaults` to save the last viewed products in the application. This dependency will be resolved at runtime using the `HelperAssembly`.

Note that we use the object scope `container` here to create a single instance of `LastViewedProductsRepositoryImplementation` in the whole application. There are several scopes available explained in more details [here](https://github.com/Swinject/Swinject/blob/master/Documentation/ObjectScopes.md).

At the end, we gather all the assemblies into an *assembler* and use it to create our dependencies.

{% highlight swift %}
class DependencyProvider {

    let container = Container()
    let assembler: Assembler

    init() {
        // the assembler gathers all the dependencies in one place
        assembler = Assembler(
            [
                HelperAssembly(),     // from low
                RepositoryAssembly(), // ...
                PresenterAssembly(),  // to high level
            ],
            container: container
        )
    }

    ...
}
{% endhighlight %}


## Custom object scopes

We have seen earlier that we have a `LastViewedProductsRepository` object, responsible for keeping a cache of the last viewed products in the app.

For now, because we used the `container` scope in the assembly, there is a single instance of this class in the whole application. And sadly enough, this instance is the same even if we log out the user, then log in with another id.

That's a bug...

We want to erase all the last viewed products when the current user logs out.

To overcome this we have a few options:
- emit a notification when the user logs out and listen to this notification in the repository
- provide a public method to clean the products (for instance `cleanCache`) in the `LastViewedProductsRepository` and call it from the outside when the user logs out.

Theses two options are fine, but not very scalable. We want to deport this logic into the DI, meaning creating a scope for the object that:
- keep the instance alive while the user is logged in
- trash the instance when the user logs out
- create a new instance when a new user logs in.

That's where custom object scopes shine.

Let's create a new object scope named `discardedWhenLogout`:

{% highlight swift %}
extension ObjectScope {
    static let discardedWhenLogout = ObjectScope(
        storageFactory: PermanentStorage.init,
        description: "discardedWhenLogout"
    )
}
{% endhighlight %}

We can use it in our assembly instead of the `container` scope:

{% highlight swift %}
class RepositoryAssembly: Assembly {

    func assemble(container: Container) {

        container.register(LastViewedProductsRepository.self) { r in
            LastViewedProductsRepositoryImplementation(
                userDefaults: r.resolve(UserDefaults.self)!
            )
        }.inObjectScope(.discardedWhenLogout) // we replace the container scope

        ...
    }
}
{% endhighlight %}

Now we can use the Swinject method `resetObjectScope(_ objectScope: ObjectScope)` on a `container`. This method discards all the instances registered in the given object scope. That means all the instances will be re-created once they are needed again.

{% highlight swift %}
func userDidLogout() {
    let container = ... // get the container somehow
    container.resetObjectScope(.discardedWhenLogout)
}
{% endhighlight %}

Now the next time a user logs in, all the last viewed products will be cleared.

## Domain specific assemblies

In some cases, there are parts of your application that you can access only if some values are downloaded, or set somewhere else in the app. In our case, let's imagine there is a configuration file downloaded when the user logs in and that this configuration is used intensively in the rest of the app.

This configuration file is mapped into a `LoginConfiguration` object, which is just a struct with a bunch of properties.

In the last viewed products page of our application, the number of products displayed is constrained with the value `maxProducts` of the configuration. We show a class of `LastViewedProductsPresenter` here, keep in mind that a *presenter* is a just like a *viewController* but not tied to UIKit.

{% highlight swift %}
class LastViewedProductsPresenterImplementation: LastViewedProductsPresenter {

    ...
    // repository to get the last viewed products
    private let lastViewedProductsRepository: LastViewedProductsRepository
    // repository to get the saved configuration
    private let loginConfigurationRepository: LoginConfigurationRepository

    init(lastViewedProductsRepository: LastViewedProductsRepository,
         loginConfigurationRepository: LoginConfigurationRepository) {
        self.lastViewedProductsRepository = lastViewedProductsRepository
        self.loginConfigurationRepository = loginConfigurationRepository
    }

    // MARK: - LastViewedProductsPresenter

    // method called when we want to reload the UI
    func reload() {
        guard let loginConfiguration = loginConfigurationRepository.savedConfiguration() else {
            return
        }
        let items = min(
            lastViewedProductsRepository.items(),
            loginConfiguration.maxProducts
        )
        // display items somehow
    }

    ...

}

{% endhighlight %}

The problem here is that we guard against the `loginConfiguration` even though the configuration **must** be available in this case, because the user must be logged in to access this page.

But the `loginConfigurationRepository` returns an optional (indeed, the configuration does not exist before the user is logged in). But if we think about it, it's not the right solution. Instead of initializing the presenter object with a repository, we should initialize it with the `loginConfiguration` directly, because we are sure at this point that such a value exists.

The `LastViewedProductsPresenterImplementation` has two dependencies, so our `PresenterAssembly` looks like this:

{% highlight swift %}
class PresenterAssembly: Assembly {

    func assemble(container: Container) {

        container.register(LastViewedProductsPresenter.self) { r in
            LastViewedProductsPresenterImplementation(
                lastViewedProductsRepository: r.resolve(LastViewedProductsRepository.self)!,
                loginConfigurationRepository: r.resolve(LoginConfigurationRepository.self)!
            )
        }

        ...
    }
}
{% endhighlight %}

What we want is to pass the `loginConfiguration` to the presenter at init time, instead of the `loginConfigurationRepository`. That means we need to store the `loginConfiguration` in the assembly itself to resolve the dependency.

Let's define a new assembly called `LoggedInPresenterAssembly`. This assembly will register types that will be used only once the user is logged in.
As the user will be logged in, we can assume the login configuration is fetched and available in the assembly:

{% highlight swift %}
class LoggedInPresenterAssembly: Assembly {

    // we store the configuration directly
    private let loginConfiguration: LoginConfiguration

    init(loginConfiguration: LoginConfiguration) {
        self.loginConfiguration = loginConfiguration
    }

    func assemble(container: Container) {

        container.register(LastViewedProductsPresenter.self) { r in
            LastViewedProductsPresenterImplementation(
                lastViewedProductsRepository: r.resolve(LastViewedProductsRepository.self)!,
                // and pass it instead of the loginConfigurationRepository
                loginConfiguration: loginConfiguration
            )
        }

        ...
    }
}
{% endhighlight %}

The only thing to do now is to create the `LoggedInPresenterAssembly` when the user logs in and that the configuration is fetched, and to give it to the assembler.

{% highlight swift %}
func userDidLogin(with configuration: LoginConfiguration) {
    let assembly = LoggedInPresenterAssembly(loginConfiguration: configuration)
    let assembler = ... // get the assembler somehow
    assembler.apply(assembly)
    ...
}
{% endhighlight %}

That way the presenter now becomes:

{% highlight swift %}
class LastViewedProductsPresenterImplementation: LastViewedProductsPresenter {

    ...
    private let lastViewedProductsRepository: LastViewedProductsRepository
    // we can use the configuration directly
    private let loginConfiguration: LoginConfiguration

    init(lastViewedProductsRepository: LastViewedProductsRepository,
         loginConfiguration: LoginConfiguration) {
        self.lastViewedProductsRepository = lastViewedProductsRepository
        self.loginConfiguration = loginConfiguration
    }

    // MARK: - LastViewedProductsPresenter

    func reload() {
        // note that the guard is gone!
        let items = min(
            lastViewedProductsRepository.items(),
            loginConfiguration.maxProducts
        )
        // display items somehow
    }

    ...

}

{% endhighlight %}

## Wrap up

Dependency injection is a major concept in programming in order to make your code reusable and testable. The concepts explained in this blog post can apply independently of the framework and the platform.

In particular, we have seen two advanced use cases:
- **custom object scopes** to customize the lifetime of your dependencies
- **domain specific assemblies** to create dependencies that depend on some external requirements


