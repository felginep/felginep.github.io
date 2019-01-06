---
layout: post
title:  "Implementing Promises in Swift"
author: "Pierre Felgines"
---

I recently have been looking for some resources on how to implement a promise in Swift, and because I did not find any good articles about it, I thought I could write one instead. In this article we'll implement our own `Promise` type in order to understand the logic behind it.

Note that the implementation is far from production ready and should not be used as is. For instance, our promise will not provide any error mechanism and threading will not be covered. I'll link useful resources and complete implementations at the end of this article for those who want to dig deeper.

*Note*: To make this tutorial a little more interesting, I chose to do it in TDD. We will write tests first and make them pass one by one.

# Our first test

Let's write our first test.

{% highlight swift %}
test(named: "0. Executor function is called immediately") { assert, done in
    var string: String = ""
    _ = Promise { string = "foo" }
    assert(string == "foo")
    done()
}
{% endhighlight %}

In this test we want to implement that the function passed to the initializer of the promise is called immediately.

*Note*: For those who may wonder, we do not use any test framework here, but a custom method `test` that simulates assertions in the playground ([gist](https://gist.github.com/felginep/039ca3b21e4f0cabb1c06126d9164680)).

When we run the playground, the compiler raises a first error:

{% highlight bash %}
error: Promise.playground:41:9: error: use of unresolved identifier 'Promise'
    _ = Promise { string = "foo" }
        ^~~~~~~
{% endhighlight %}

Fair enough, we need to define the `Promise` class.

{% highlight swift %}
class Promise {

}
{% endhighlight %}

The error now becomes:

{% highlight bash %}
error: Promise.playground:44:17: error: argument passed to call that takes no arguments
    _ = Promise { string = "foo" }
                ^~~~~~~~~~~~~~~~~~
{% endhighlight %}

We have to define an initiliazer that takes a closure as an argument. And this closure should be called immediately.

{% highlight swift %}
class Promise {

    init(executor: () -> Void) {
        executor()
    }
}
{% endhighlight %}

This makes our first test pass! We have almost nothing for now, but be patient, our implementation will grow in the next section!

{% highlight bash %}
• Test 0. Executor function is called immediately passed (1 assertions)
{% endhighlight %}

We can comment this test out as the implementation of `Promise` will change a bit in the future.

# The bare minimum

Our second test is the following:

{% highlight swift %}
test(named: "1.1 Resolution handler is called when promise is resolved sync") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        resolve(string)
    }
    promise.then { (value: String) in
        assert(string == value)
        done()
    }
}
{% endhighlight %}

The test is quite simple, but we add some content to the `Promise` class. We create a promise with a resolution handler (the `resolve` parameter of the closure) and call it right away with a value.
In a second time, we use the `then` method on the promise to access the value and make an assertion about it.

Before diving into the implementation, we have to introduce a slightly different test at the same time.

{% highlight swift %}
test(named: "1.2 Resolution handler is called when promise is resolved async") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        after(0.1) {
            resolve(string)
        }
    }
    promise.then { (value: String) in
        assert(string == value)
        done()
    }
}
{% endhighlight %}

Contrary to the test `1.1`, the `resolve` method is called after a delay. That means the value will not be available in the `then` right away (because the 0.1 seconds are not passed yet when we call `then`).

We can start understanding the "problem" here. We have to deal with asynchronicity.

Our promise is a state machine. When it is first created, the promise is in a `pending` state. Once the `resolve` method is called with a value, our promise pass in a `resolved` state and store this value.

The method `then` can be called anytime, whatever the internal state of the promise (meaning before or after the promise has a value). If the promise is in `pending` state and we call `then` on it, the value is not available, so we have to store the callback parameter. And once the promise becomes `resolved`, we can trigger this same callback with the resolved value.

Now that we understand a little better what we have to implement, let's start by fixing the compiler issues.

{% highlight bash %}
error: Promise.playground:54:19: error: cannot specialize non-generic type 'Promise'
    let promise = Promise<String> { resolve in
                  ^      ~~~~~~~~
{% endhighlight %}

We have to make our `Promise` type generic. Indeed, a promise is associated with a predefined type and will hold a value of this type once resolved.

{% highlight swift %}
class Promise<Value> {

    init(executor: () -> Void) {
        executor()
    }
}
{% endhighlight %}

Now the error becomes:

{% highlight bash %}
error: Promise.playground:54:37: error: contextual closure type '() -> Void' expects 0 arguments, but 1 was used in closure body
    let promise = Promise<String> { resolve in
                                    ^
{% endhighlight %}

We have to provide a `resolve` function to the closure passed in the initializer (the executor).

{% highlight swift %}
class Promise<Value> {

    init(executor: (_ resolve: (Value) -> Void) -> Void) {
        executor()
    }
}
{% endhighlight %}

Note here that the resolve parameter is a function that consume a value: `(Value) -> Void`. This function will be called by the outside world once the value is determined.

The compiler is still not happy because we need to provide a `resolve` function to the `executor`. Let's create one that will be `private`.

{% highlight swift %}
class Promise<Value> {

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    private func resolve(_ value: Value) -> Void {
        // To implement
        // This will be called by the outside world when a value is determined
    }
}
{% endhighlight %}

We will implement `resolve` in a moment, when all the errors will be taken care of.

The next one is simple, the method `then` is not defined yet.

{% highlight bash %}
error: Promise.playground:61:5: error: value of type 'Promise<String>' has no member 'then'
    promise.then { (value: String) in
    ^~~~~~~ ~~~~
{% endhighlight %}

Let's fix that.

{% highlight swift %}
class Promise<Value> {

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    func then(onResolved: @escaping (Value) -> Void) {
        // To implement
    }

    private func resolve(_ value: Value) -> Void {
        // To implement
    }
}
{% endhighlight %}

Now that the compiler is happy, let's go back where we were before.

We previously said that a `Promise` is a state machine with a `pending` and a `resolved` state. We can define these states with an enum:

{% highlight swift %}
enum State<T> {
    case pending
    case resolved(T)
}
{% endhighlight %}

The beauty of Swift makes it possible to store the value of the promise directly in the enum.

Now we have to define a default state of `.pending` in our `Promise` implementation. And we also need a private function that can update the state in case the promise is still in a `.pending` state.

{% highlight swift %}
class Promise<Value> {

    enum State<T> {
        case pending
        case resolved(T)
    }

    private var state: State<Value> = .pending

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    func then(onResolved: @escaping (Value) -> Void) {
        // To implement
    }

    private func resolve(_ value: Value) -> Void {
        // To implement
    }

    private func updateState(to newState: State<Value>) {
        guard case .pending = state else { return }
        state = newState
    }
}
{% endhighlight %}

Note that the `updateState(to:)` function first checks if the promise is in the `.pending` state. If the promise is already in the `.resolved` state, it can't move to another state and will stay `.resolved` forever.

Now it's time to update the state of the promise when needed, meaning when the `resolve` function is called from the outside world with a value.

{% highlight swift %}
private func resolve(_ value: Value) -> Void {
    updateState(to: .resolved(value))
}
{% endhighlight %}

We are almost done, but there is still the `then` method to implement. We said we had to store the callback parameter and call this callback when the promise resolves. Let's implement that.

{% highlight swift %}
class Promise<Value> {

    enum State<T> {
        case pending
        case resolved(T)
    }

    private var state: State<Value> = .pending
    // we store the callback as an instance variable
    private var callback: ((Value) -> Void)?

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    func then(onResolved: @escaping (Value) -> Void) {
        // store the callback in all cases
        callback = onResolved
        // and trigger it if needed
        triggerCallbackIfResolved()
    }

    private func resolve(_ value: Value) -> Void {
        updateState(to: .resolved(value))
    }

    private func updateState(to newState: State<Value>) {
        guard case .pending = state else { return }
        state = newState
        triggerCallbackIfResolved()
    }

    private func triggerCallbackIfResolved() {
        // the callback can be triggered only if we have a value,
        // meaning the promise is resolved
        guard case let .resolved(value) = state else { return }
        callback?(value)
        callback = nil
    }
}
{% endhighlight %}

We define an instance variable `callback` that holds the callback while the promise is `.pending`. We also create a method `triggerCallbackIfResolved` that first checks if the state is `.resolved`, unwraps the associated value, and pass it to the callback. This method is called at two locations. In the `then` method if the promise is already resolved at the time we call `then`. And in the `updateState` method, because that's where the promise updates its state from `.pending` to `.resolved`.

With these modifications, our tests pass successfully.

{% highlight bash %}
• Test 1.1 Resolution handler is called when promise is resolved sync passed (1 assertions)
• Test 1.2 Resolution handler is called when promise is resolved async passed (1 assertions)
{% endhighlight %}

We are on the right path, but there is still a slight change we have to make to get a first real `Promise` implementation. Let's look at the tests first.

{% highlight swift %}
test(named: "2.1 Promise supports many resolution handlers sync") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        resolve(string)
    }
    promise.then { value in
        assert(string == value)
    }
    promise.then { value in
        assert(string == value)
        done()
    }
}
{% endhighlight %}

{% highlight swift %}
test(named: "2.2 Promise supports many resolution handlers async") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        after(0.1) {
            resolve(string)
        }
    }
    promise.then { value in
        assert(string == value)
    }
    promise.then { value in
        assert(string == value)
        done()
    }
}
{% endhighlight %}

This time we call `then` twice on the promise.

Let's look at the tests outputs.

{% highlight bash %}
• Test 2.1 Promise supports many resolution handlers sync passed (2 assertions)
• Test 2.2 Promise supports many resolution handlers async passed (1 assertions)
{% endhighlight %}

The tests pass, but you may have spotted the issue. The test `2.2` has only one assertion, but should have two.

If we think about it, it's logical. Indeed, in the async version (`2.2`), when the first `then` is called the promise is still `.pending`. As we have seen earlier, we store the callback of the first `then`. But when we call the `then` for the second time, the promise hasn't resolved yet and is still `.pending`, so we erase the callback with the new one. Only the second callback will be executed, the first one being forgotten. That makes the test pass, but with only one assertion instead of two.

The solution here is to store an array of callbacks and to trigger all the callbacks when the promise resolves.

Let's update our solution with this small update.

{% highlight swift %}
class Promise<Value> {

    enum State<T> {
        case pending
        case resolved(T)
    }

    private var state: State<Value> = .pending
    // We now store an array instead of a single function
    private var callbacks: [(Value) -> Void] = []

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    func then(onResolved: @escaping (Value) -> Void) {
        callbacks.append(onResolved)
        triggerCallbacksIfResolved()
    }

    private func resolve(_ value: Value) -> Void {
        updateState(to: .resolved(value))
    }

    private func updateState(to newState: State<Value>) {
        guard case .pending = state else { return }
        state = newState
        triggerCallbacksIfResolved()
    }

    private func triggerCallbacksIfResolved() {
        guard case let .resolved(value) = state else { return }
        // We trigger all the callbacks
        callbacks.forEach { callback in callback(value) }
        callbacks.removeAll()
    }
}
{% endhighlight %}

The tests now pass with both two assertions.

{% highlight bash %}
• Test 2.1 Promise supports many resolution handlers sync passed (2 assertions)
• Test 2.2 Promise supports many resolution handlers async passed (2 assertions)
{% endhighlight %}

Congratulations! We have created the base of our `Promise` class. You can already use it to abstract  asynchronicity but it's still limited.

**Note**: if we take a look at the global picture here, we can see that the `then` we have defined could be renamed `observe`. Its purpose is to consume the value of the promise once resolved, but it does not return anything. Meaning we can't chain promises for now.

In the next sections we will create overloads of `then` in order to return new promises or new values along the way.

# Chaining Promises

Our `Promise` implementation would not be complete if we can't chain multiple promises.

Let's look at the test that will help us implement this feature.

{% highlight swift %}
test(named: "3. Resolution handlers can be chained") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        after(0.1) {
            resolve(string)
        }
    }
    promise
        .then { value in
            return Promise<String> { resolve in
                after(0.1) {
                    resolve(value + value)
                }
            }
        }
        .then { value in // the "observe" previously defined
            assert(string + string == value)
            done()
        }
}
{% endhighlight %}

We can see here that the first `then` creates a new `Promise` with a whole new value and returns it. The second `then`(the one we defined in the previous section, that we called `observe`) is chained to access the new value (that will hold `"foofoo"` here).

This immediately raises an error in the console.

{% highlight bash %}
error: Promise.playground:143:10: error: value of tuple type '()' has no member 'then'
        .then { value in
         ^
{% endhighlight %}

We have to create an overload of `then` that takes a function that returns a promise. And in order to chain other calls of `then`, the method has to return a promise too. The prototype of this new method `then` is the following.

{% highlight swift %}
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    // to implement
}
{% endhighlight %}

*Note*: Attentive readers may have spotted that we are implementing `flatMap` on the `Promise` type here. In the same way `flatMap` is defined for `Optional` or `Array`, we can define it for the `Promise` type.

The "difficulty" starts here. Let's walk through the implementation of this "flatMap" `then` step by step.

- We have to return a `Promise<NewValue>`.
- What gives us such a promise? The `onResolved` method does.
- But `onResolved` takes a value of type `Value` in parameter. How can we get this value? We can use the previously defined `then` (or "`observe`") to access it when available.

If we write this down, here what we get for now:

{% highlight swift %}
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    then { value in // the "observe" one
        let promise = onResolved(value) // `promise` is a Promise<NewValue>
        // problem: how do we return `promise` to the outside ??
    }
    return // ??!!
}
{% endhighlight %}

We are almost there. There is still a small problem to fix: the `promise` variable is captured in the closure passed to `then`. We can't use it as a return value for the function.

The trick here is to create a wrapping `Promise<NewValue>` that will execute what we wrote so far, and that will resolve at the same time the `promise` variable is resolved. In other words, when the promise provided by the `onResolved` method will resolve and get a value from the outside, the wrapping promise will resolve too and get the same value.

That may be a little abstract, but if we write it, we will see it better:

{% highlight swift %}
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    // We have to return a promise, so let's return a new one
    return Promise<NewValue> { resolve in
        // this is called immediately as seen in test 0.
        then { value in // the "observe" one
            let promise = onResolved(value) // `promise` is a Promise<NewValue>
            // `promise` has the same type of the Promise wrapper
            // we can make the wrapper resolves when the `promise` resolves
            // and gets a value
            promise.then { value in resolve(value) }
        }
    }
}
{% endhighlight %}

If we clean the code a bit we now have these two methods :

{% highlight swift %}
// observe
func then(onResolved: @escaping (Value) -> Void) {
    callbacks.append(onResolved)
    triggerCallbacksIfResolved()
}

// flatMap
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    return Promise<NewValue> { resolve in
        then { value in
            onResolved(value).then(onResolved: resolve)
        }
    }
}
{% endhighlight %}

And finally, the test pass.

{% highlight bash %}
• Test 3. Resolution handlers can be chained passed (1 assertions)
{% endhighlight %}

# Chaining values

If you can implement `flatMap` on a type, you can implement `map` on this same type using `flatMap`. What does `map` looks like for our `Promise` ?

The test we will use here is the following:

{% highlight swift %}
test(named: "4. Chaining works with non promise return values") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        after(0.1) {
            resolve(string)
        }
    }
    promise
        .then { value -> String in
            return value + value
        }
        .then { value in // the "observe" then
            assert(string + string == value)
            done()
        }
}
{% endhighlight %}

Note here that the first `then` that is used does not return a `Promise` anymore, but transforms the value it receives. This is a new `then` and corresponds to the `map` version we want to add.

The compiler emits an error saying we have to implement this method.

{% highlight bash %}
error: Promise.playground:174:26: error: declared closure result 'String' is incompatible with contextual type 'Void'
        .then { value -> String in
                         ^~~~~~
                         Void
{% endhighlight %}

The prototype is very close to the `flatMap` version, the only difference is that we return a `NewValue` instead of a `Promise<NewValue>` in the `onResolved` function.

{% highlight swift %}
// map
func then<NewValue>(onResolved: @escaping (Value) -> NewValue) -> Promise<NewValue> {
    // to implement
}
{% endhighlight %}

We said earlier that we can use `flatMap` to implement `map`. In our case we see we need to return a `Promise<NewValue>`. If we use the "flatMap" `then` of the previous section and create a promise that directly resolved with the mapped value, we have finished. Let's prove it.

{% highlight swift %}
// map
func then<NewValue>(onResolved: @escaping (Value) -> NewValue) -> Promise<NewValue> {
    return then { value in // the "flatMap" defined before
        // must return a Promise<NewValue> here
        // this promise directly resolves with the mapped value
        return Promise<NewValue> { resolve in
            let newValue = onResolved(value)
            resolve(newValue)
        }
    }
}
{% endhighlight %}

Once again the test pass.

{% highlight bash %}
• Test 4. Chaining works with non promise return values passed (1 assertions)
{% endhighlight %}

If we remove the comments and look at what we achieved, we have three `then` methods implemented that can be used and chained.

{% highlight swift %}
// observe
func then(onResolved: @escaping (Value) -> Void) {
    callbacks.append(onResolved)
    triggerCallbacksIfResolved()
}

// flatMap
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    return Promise<NewValue> { resolve in
        then { value in
            onResolved(value).then(onResolved: resolve)
        }
    }
}

// map
func then<NewValue>(onResolved: @escaping (Value) -> NewValue) -> Promise<NewValue> {
    return then { value in
        return Promise<NewValue> { resolve in
            resolve(onResolved(value))
        }
    }
}
{% endhighlight %}

# Example of use

We will stop here for the implementation. Our `Promise` class is complete enough to demonstrate what we can do with it.

Let's imagine we have some users in our app, that we store in the following struct:

{% highlight swift %}
struct User {
    let id: Int
    let name: String
}
{% endhighlight %}

Let's also say we have two methods at our disposable, one that fetch a list of user ids, and one that fetch a user from its id. And let's say we want to display the name of the first user.

Here is how we can do it very simply with our implementation, using the three `then` we previously defined.

{% highlight swift %}
func fetchIds() -> Promise<[Int]> {
    ...
}

func fetchUser(id: Int) -> Promise<User> {
    ...
}

fetchIds()
    .then { ids in // flatMap
        return fetchUser(id: ids[0])
    }
    .then { user in // map
        return user.name
    }
    .then { name in // observe
        print(name)
    }

{% endhighlight %}

The code becomes highly readable, flat and concise!

# Conclusion

That's the end of this article, I hope you liked it.

You can find the whole code in this [gist](https://gist.github.com/felginep/039ca3b21e4f0cabb1c06126d9164680). And if you want to dig deeper, here are the sources I used.

- [Promises in Swift by Khanlou](http://khanlou.com/2016/08/promises-in-swift/)
- [JavaScript Promises ... In Wicked Detail](https://www.mattgreer.org/articles/promises-in-wicked-detail/)
- [PromiseKit 6 Release Details](https://promisekit.org/news/2018/02/PromiseKit-6.0-Released/)
- [TDD Implementation of Promises in JavaScript](https://www.youtube.com/watch?v=C3kUMPtt4hY)
