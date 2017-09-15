---
layout: post
title:  "Coordinators and Back Button"
author: "Pierre Felgines"
---

This article was first published on Fabernovel [blog](https://en.fabernovel.com/insights/tech-en/coordinators-and-back-button).

To build modulable and easily maintainable iOS applications, we decided to use, at Applidium, a custom iOS architecture, adapted from the Uncle Bob’s [clean architecture](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html). One of the biggest challenge, in that process, was to separate the view, business and navigation logics from view controllers. The view and the business logics are isolated and decoupled from the view controllers thanks to view contracts, presenters, interactors and repositories, mostly. The navigation part is done through [Coordinators](http://khanlou.com/2015/01/the-coordinator/). Coordinators take care of presenting and dismissing view controllers, handle the application workflow and are organized in a tree hierarchy standing in parallel to the view controller’s one.

# Coordinators grow with the application

At first, there is generally one coordinator per navigation controller. Pushed view controllers are handled by the same coordinator, while modally presented ones live in a new navigation controller supervised by a child coordinator. The parent coordinator is in charge to present and dismiss this new navigation controller.

As the application complexity increases, coordinators handle more and more logic, mostly when screens are accessible from different places. This leads to coordinators implementing a lot of navigation delegation protocols (schema 1).

![schema_1](https://www.fabernovel.com/content/uploads/2017/09/Coordinator.png "Schema 1. One coordinator handle multiple logically related units"){: .center-image }
*Schema 1. One coordinator handle multiple logically related units*

When a coordinator handle too many view controllers in a navigation controller, it’s a good idea to refactor and to break the logic in multiple coordinators, each one handling a subset of the navigation controller stack. All those new coordinators are dispatched one after the other and share the same navigationController, each of them taking care of a distinct logic unit (schema 2).

![schema_2](https://www.fabernovel.com/content/uploads/2017/09/CoordinatorRefactor.png "Schema 2. Multiple coordinators handle logically related units"){: .center-image }
*Schema 2. Multiple coordinators handle logically related units*

As stated in [this article](http://khanlou.com/2017/05/back-buttons-and-coordinators/), this leads to issues when the user taps the back button because the action is handled by the system and the back button was not created in our code. It’s then difficult to track which coordinator should be alive and to proceed to our regular bookkeeping. In the same article, Soroush Khanlou provides some interesting solutions about this problem, that we will challenge before proposing another solution that better suits our needs.

# Available solutions

[Bryan Irace’s](http://irace.me/navigation-coordinators) solution, that Soroush is presenting, uses a custom view controller containing and handling the navigation controller. Having another container view controller increases both the view controller and the view hierarchies. This also prevents the navigation controller pop gesture recognizer from working properly, requiring to use its delegate to put it back on track.

Moreover the solution let view controllers be aware of coordinators. This breaks our architecture, as coordinators should take control of navigation on top of view controllers.
To avoid that tradeoff, Soroush decided to move the navigation controller delegate to the coordinator. But anyway, there is still some issues down there:

- there is an additional view controller container
- the use of the gesture recognizer delegate is visible from the outside, it means we can not use it anywhere else without breaking the mechanism
- same problem happens with the navigation controller delegate, the spot is taken, we cannot add another one (for a custom transition for example)
- as the first coordinator is the navigation controller’s delegate, this become complicated to stack more than two coordinators for the same navigation controller

This is why we decided to work on our proposition to deal with those issues. The idea was to build a mechanism easily and optionally usable that is as invisible as possible from the outside of the “I need to stack coordinators for a single navigation controller” scope.

# Our solution

To sum up, here are our requirements:

- be decoupled from coordinators and UI code
- track which view controller has been popped from the navigation controller
- be able to change the view controller stack in the different coordinators
- allow the use of custom transitions for push / pop operations

Let’s see the final API in a simple example, before digging into the details.


{% highlight swift %}
protocol DetailCoordinatorDelegate : class {
    func detailCoordinatorDidDismiss(coordinator: DetailCoordinator)
}

class DetailCoordinator : Coordinator, NavigationControllerObserverDelegate {

    weak var delegate: DetailCoordinatorDelegate?

    private let navigationController: UINavigationController
    private let navigationControllerObserver: NavigationControllerObserver

    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
        self.navigationControllerObserver = NavigationControllerObserver(
            navigationController: navigationController
        )
        super.init()
    }

    //MARK: - Coordinator

    func start(animated: Bool) {
        let detailViewController = DetailViewController()
        navigationController.pushViewController(detailViewController, animated: animated)
        navigationControllerObserver.observePopTransition(
            of: detailViewController,
            delegate: self
        )
    }

    //MARK: - NavigationControllerObserverDelegate

    func navigationControllerObserver(_ observer: NavigationControllerObserver,
                                      didObservePopTransitionFor viewController: UIViewController) {
        delegate?.detailCoordinatorDidDismiss(coordinator: self)
    }
}
{% endhighlight %}

The interesting part is the line `navigationControllerObserver.observePopTransition(of: detailViewController, delegate: self)`.

Indeed the `DetailCoordinator` is in charge to create the detail view controller, to push it into its graphic context (the navigation controller) and to notify its parent that it has been dismissed.

The `navigationControllerObserver` has a method to observe the pop of the detailViewController so that we can be notified.

The first thing to do when we want to track navigation controller operations is to use its `delegate` property.

{% highlight swift %}
public class NavigationControllerObserver : NSObject, UINavigationControllerDelegate {

    public init(navigationController: UINavigationController) {
        self.navigationController = navigationController
        super.init()
        navigationController.delegate = self
    }

    ... 
}
{% endhighlight %}

Then we want to register a delegate to a specific view controller to observe. When a view controller is popped, we will find its associated delegate, and if it exists, notify it of the transition.

{% highlight swift %}
public class NavigationControllerObserver : NSObject, UINavigationControllerDelegate {

    private var viewControllersToDelegates: [UIViewController: NavigationControllerObserverDelegateContainer] = [:]
    
    public func observePopTransition(of viewController: UIViewController,
                                     delegate: NavigationControllerObserverDelegate) {
        let wrappedDelegate = NavigationControllerObserverDelegateContainer(value: delegate)
        viewControllersToDelegates[viewController] = wrappedDelegate
    }
    
    //MARK: - UINavigationControllerDelegate

    public func navigationController(_ navigationController: UINavigationController,
                                     didShow viewController: UIViewController,
                                     animated: Bool) {

        guard let fromViewController = navigationController.transitionCoordinator?.viewController(forKey: .from),
            !navigationController.viewControllers.contains(fromViewController) else {
                return
        }

        if let wrappedDelegate = viewControllersToDelegates[fromViewController] {
            let delegate = wrappedDelegate.value
            delegate?.navigationControllerObserver(self, didObservePopTransitionFor: fromViewController)
            viewControllersToDelegates.removeValue(forKey: fromViewController)
        }
    }   
}
{% endhighlight %}

The first thing to note here is that we need to wrap the delegate into a `NavigationControllerObserverDelegateContainer` to store a weak value and avoid a retain cycle.

{% highlight swift %}
public protocol NavigationControllerObserverDelegate : class {
    func navigationControllerObserver(_ observer: NavigationControllerObserver,
                                      didObservePopTransitionFor viewController: UIViewController)
}

// We can't use Weak<T: AnyObject> here because we would have
// to declare NavigationControllerObserverDelegate as @objc
private class NavigationControllerObserverDelegateContainer {

    private(set) weak var value: NavigationControllerObserverDelegate?

    init(value: NavigationControllerObserverDelegate) {
        self.value = value
    }
}
{% endhighlight %}

In the `navigationController(_:didShow:animated)` callback, we first get the `fromViewController` from the transition, and verify that it is not in the navigationController’s stack anymore. That means we are in a pop transition and not in a push.

Then we find the delegate to call and let it know about the transition.

That’s all it takes to register multiple delegates, each per view controller. For now we have achieved the followings:

- be decoupled from coordinators and UI code. Our `navigationControllerObserver` knows nothing about the architecture of the project, about coordinators, or about view controllers. Its only job is to notify the right delegate when a pop occurs.
- track which view controller has been popped from navigation controller. This is done through the use of the `UINavigationControllerDelegate` methods.
- be able to change the view controller stack in the different coordinators. Indeed the parent coordinator can change the stack it owns with no consequences. Each time the detail coordinator changes the first view controller of the navigation controller’s stack, it also needs to observe the new controller pop transition by calling `observePopTransition(of:delegate)`.

# Objective-C to the rescue

The last requirement we got, was to allow the use of custom transitions for push or pop operations. These custom transitions are handled by the navigation controller delegate, that is already in use in the `NavigationControllerObserver`.

To make it available to the outside world, we create a property `navigationControllerDelegate` and forward all `UINavigationControllerDelegate` methods to this object with some Objective-C wizardry.

{% highlight swift %}
public class NavigationControllerObserver : NSObject, UINavigationControllerDelegate {

    public weak var navigationControllerDelegate: UINavigationControllerDelegate? {
        didSet {
            // Make the navigationController reevaluate respondsToSelector
            // for UINavigationControllerDelegate methods
            navigationController.delegate = nil
            navigationController.delegate = self
        }
    }

    //MARK: - NSObject

    override public func responds(to aSelector: Selector!) -> Bool {
        if shouldForwardSelector(aSelector) {
            return navigationControllerDelegate?.responds(to: aSelector) ?? false
        }
        return super.responds(to: aSelector)
    }

    override public func forwardingTarget(for aSelector: Selector!) -> Any? {
        if shouldForwardSelector(aSelector) {
            return navigationControllerDelegate
        }
        return super.forwardingTarget(for: aSelector)
    }

    //MARK: - UINavigationControllerDelegate

    public func navigationController(_ navigationController: UINavigationController,
                                     didShow viewController: UIViewController,
                                     animated: Bool) {
        navigationControllerDelegate?.navigationController?(
            navigationController,
            didShow: viewController,
            animated: animated
        )
        ...
    }

    //MARK: - Private

    private func shouldForwardSelector(_ aSelector: Selector!) -> Bool {
        let description = protocol_getMethodDescription(UINavigationControllerDelegate.self, aSelector, false, true)
        return
            description.name != nil // belongs to UINavigationControllerDelegate
                && class_getInstanceMethod(type(of: self), aSelector) == nil // self does not implement aSelector
    }
}
{% endhighlight %}

We can argue that using Objective-C dynamic features is not the right solution. For our needs it works pretty well because it stays hidden in this particular object, and allows the outside world to provide navigation controller delegate method implementations.

# What did we accomplish ?

Finally, let’s see the code in the `MasterCoordinator` that creates the `DetailCoordinator`.

{% highlight swift %}
class MasterCoordinator : Coordinator, MasterViewControllerDelegate, DetailCoordinatorDelegate {
    
    ...
    
    //MARK: - MasterViewControllerDelegate

    func masterViewControllerDidRequestDetail() {
        let detailCoordinator = DetailCoordinator(
            navigationController: navigationController
        )
        detailCoordinator.delegate = self
        addChild(detailCoordinator)
        detailCoordinator.start(animated: true)
    }

    //MARK: - DetailCoordinatorDelegate

    func detailCoordinatorDidDismiss(coordinator: DetailCoordinator) {
        // The important line
        removeChild(coordinator)
    }
}
{% endhighlight %}

All our work has been done to have a chance to call `removeChild(coordinator)` and release the coordinator that is not in use anymore.

Here is a visual representation of the different objects in play:

![schema_3](https://www.fabernovel.com/content/uploads/2017/09/NavigationObserver.png "Schema 3. Object interactions"){: .center-image }
*Schema 3. Object interactions*

Our solution is not a perfect one, but we take a slightly different approach from the two other solutions that were proposed in the Soroush’s [article](http://khanlou.com/2017/05/back-buttons-and-coordinators/). We hope these series of articles will help people use Coordinators, because it’s a great tool to extract navigation logic out of view controllers. You can find the full gist [here](https://gist.github.com/felginep/370131d12a5ced76accd109e731e6d76).

A special thanks to **Benjamin Lavialle**, who helped and contributed to this blog post.
