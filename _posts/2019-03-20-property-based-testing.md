---
layout: post
title: "Property based testing"
author: "Pierre Felgines"
---

You will always hear that writing tests is a good thing. But are you 100% sure about the pertinence of your test suite? What if you test only edges cases and miss something?
Property based testing let you generate your tests instead of writing them.
Let's see what this is, how to use it, and how we can leverage it to write robust UI tests.

# Basic test

Let's say for the sake of the demonstration that we want to implement a sort algorithm in our application.

Our implementation looks like this:

{% highlight swift %}
extension Array where Element: Comparable {

    func quickSort() -> [Element] {
        return quickSort(self)
    }

    private func quickSort(_ array: [Element]) -> [Element] {
        if array.count < 3 { return array }
        var data = array
        let pivot = data.remove(at: 0)
        let left = data.filter { $0 < pivot }
        let right = data.filter { $0 >= pivot }
        return quickSort(left) + [pivot] + quickSort(right)
    }
}
{% endhighlight %}

If we want to test our code, we could write some example based unit tests like so:

{% highlight swift %}
XCTAssertEqual([1, 2, 4, 5, 3].quickSort(), [1, 2, 3, 4, 5])
XCTAssertEqual([1].quickSort(), [1])
XCTAssertEqual([].quickSort(), [])
{% endhighlight %}

Attentive readers may have spotted that our code is buggy, but our tests pass anyway. How could we test it better to find the bug?

# Property based testing

We wrote some tests, that's a very good thing, but our test suite is really minimal. In a perfect world, we would like to test the `quickSort()` function for every array of integers.

What we call a *property based test* is a test that assert a certain property holds for every input we use (meaning every array of integers in our case).

Conceptually, here is a property based test:
{% highlight swift %}
// repeat 100 times
// generate random input
for all (x, y, ...)
// keep generating inputs while this is false
such that precondition(x, y, ...) holds
// test the property
property(x, y, ...) is true
{% endhighlight %}

In our example, the property is rather simple: if we use `quickSort()` on an array, the array should be sorted (meaning every element is lower or equal than the next element).

In pseudo code, here is our property based test.

{% highlight swift %}
forAll array of integers:
    array.quickSort().isSorted()
    // the `isSorted` function compares pair of elements
{% endhighlight %}

In practice, even if this is not possible to test *every* array of integer in the world, we can write tests that run this property for *a lot* of arrays. And this will be more powerful that the first example based tests we wrote earlier.

The most known library to do property based testing is called [QuickCheck](http://hackage.haskell.org/package/QuickCheck) in Haskell. In Swift, we can use [SwiftCheck](https://github.com/typelift/SwiftCheck) instead.

Here is the protocol at the core of the library:

{% highlight swift %}
public protocol Arbitrary {
    // a random generator of Self
    static var arbitrary: Gen<Self> { get }
    // a function that returns smaller instances of Self than the current one
    static func shrink(_ : Self) -> [Self]
}
{% endhighlight %}

In essence, this protocol is composed of two components:
- a random value generators (primitive types already conform to `Arbitrary`)
- a way to shrink a value from an instance

I encourage you to go read the tutorial [playground](https://github.com/typelift/SwiftCheck/tree/master/Tutorial.playground) if you want to well understand the concepts.

The random value generator is useful to generate a lot of random inputs, but what do we call *shrinking*?

When SwiftCheck finds a test that fails for a random input, it will not simply return the input, but will iterate on some *shrunk* values of this input to find the minimal input that make the test fail. By definition, shrunk values have a size less or equal than the initial value.

For instance, if we ask the shrunk values for the `"test"` string, we will get this result:

{% highlight swift %}
print(String.shrink("test"))

["t", "st", "te", "est", "tst", "tet", "tes", " est", "2est", "best",
"cest", "Cest", "aest", "1est", "\nest", "Aest", "Best", "3est",
"t st", "t2st", "tbst", "tcst", "tCst", "tast", "t1st", "t\nst",
"tBst", "tAst", "t3st", "te2t", "tebt", "tect", "teCt", "teat",
"te1t", "te\nt", "teBt", "teAt", "te3t", "te t", "tes ", "tes2",
"tesb", "tesc", "tesC", "tesa", "tes1", "tes\n", "tesA", "tesB", "tes3"]
{% endhighlight %}

# Back to the example

Now that we understand how to implement a property based test, let's do it for the `quickSort()` function we defined earlier.

Following the SwiftCheck syntax, here is the test:
{% highlight swift %}
import SwiftCheck

property("Sort integers") <- forAll { (integers: [Int]) in
    return integers.quickSort().isSorted
}
{% endhighlight %}

When we run this code, we get an error right away !

{% highlight swift %}
[]
[1]
[-1, 0]
[]
[-3, 0, 2, 2]
[]
[-5, 1, 5, 4]
*** Failed! [4]
[5, 4]
[-5, 1]
[1, 5, 4]
[-5, 5, 4]
[-5, 1, 4]
[-5, 1, 5]
// ... Test ~600 more values to find minimal error
[1, 0]
[0, 0]
[0, 1]
[0, 0, 0]
[]
[0]
[1]
[0, 0]
Proposition: Test
Falsifiable (after 7 tests and 11 shrinks):
[1, 0]
*** Passed 6 tests
.
{% endhighlight %}

Our tests were not that strong because we already have found a failing input... In our case this is the array `[1, 0]`.

Note that the test did fail first with the input `[-5, 1, 5, 4]`. Then the library did shrink the value `[-5, 1, 5, 4]` to find the minimal input that makes our property fail. It generated more than 600 different values (7 tests and 11 shrinks) to find that `[1, 0]` is the minimal input that makes the test fail.

We can easily spot the bug with the failing input.

{% highlight swift %}
private func quickSort(_ array: [Element]) -> [Element] {
    if array.count < 2 { return array } // replace 3 with 2
    ...
}
{% endhighlight %}

This time the result is correct, all the 100 tests pass.

{% highlight swift %}
*** Passed 100 tests
.
{% endhighlight %}

# When to use property based testing?

The advantages of property based testing are:
- they are more general and can replace many example based tests
- they test all the edge cases (null, negative numbers, empty arrays, uncommon strings)
- they are reproducible (once a test fail, we know for which input, and we can rerun the test)
- they shrink the input in case of a failure

But that does not mean you have to remove all your example based tests and replace them with property based tests. In practice it can be really difficult to write property based tests, and example based tests are important because they are simple. Meaning that someone else can quickly understand their utility.

There are some special cases where property based testing shines. For instance, when you have two symmetrical functions (for instance an `Encoder` and a `Decoder`): for any input, the result of passing the input in the first function, then in the reverse function should be equal to the initial input.
This idea is well covered in the [talk](https://www.youtube.com/watch?v=ME9aYZ9qGHQ) of Jack Flintermann, creator of the [Dwiff](https://github.com/jflinter/Dwifft) library that explains that property based testing helped him catch some weird bugs during refactoring.

For example, let's say we want have a function that sub divide an array into chunks of a size `n`:

{% highlight swift %}
extension Array {
    func chunked(into size: Int) -> [[Element]] {
        guard size > 0 else { return [] }
        return stride(from: 0, to: count, by: size).map {
            Array(self[$0 ..< Swift.min($0 + size, count)])
        }
    }
}
{% endhighlight %}

The reverse function is easy to find, we just need to reassemble all the arrays to get the input back.

The test associated with this function could be:

{% highlight swift %}
// A generator for strictly positive integers
let sizeGen = Int.arbitrary.suchThat { $0 > 0 }
// The default generator for array of integers
let arrayGen = Array<Int>.arbitrary

property("Test Chunk") <- forAll(arrayGen, sizeGen) { (integers: [Int], size: Int) in
    return integers.chunked(into: size).reduce([], +) == integers
}
{% endhighlight %}

# What about UI tests?

As we mainly write code that is related to UI, how could we leverage property based testing to test our layouts? For instance, it's very common to forget activating an `NSLayoutConstraint`, or to provide the wrong `constant` value, and this results in layout issues. And sometimes these issues do not appear during the development phase, but later, in production with real data.

To use property based testing, the idea is similar to our previous example. We need to generate random inputs and find a property that holds for our layout for every input.

The property is easy to find: a view is just a bunch of subviews, and we want to assert that our layout is correct when there are no views overlap, no autolayout errors and no clipped subviews.

Now, how to generate our inputs?

I always create a struct called a `ViewModel` to configure my views. This is just a dumb data structure that holds all the properties that will be displayed in the view (booleans to hide / show subviews, strings to display in labels, etc.).

So all I need is to generate random view models, pass them to my view, and then assert that the layout is correct.

Here is an example of how to generate a random view model:

{% highlight swift %}
// A view model that could configure a profile page
struct ViewModel {
    let userName: String
    let messagesCount: Int
    let displayFullProfile: Bool
}

extension ViewModel: Arbitrary {

    // This is a ViewModel random generator
    static var arbitrary: Gen<ViewModel> {
        // We create a lower string random generator
        let lowerString = Gen<Character>
            // use a random value between "a" and "z"
            .fromElements(in: "a"..."z")
            // create an array of random length of these characters
            .proliferateNonEmpty
            // create a String from this array of characters
            .map { String.init($0) }.
        return Gen<ViewModel>.compose { c in
            ViewModel(
                // use our lower string random generator
                userName: c.generate(using: lowerString),
                // use a positive integer random generator
                messagesCount: c.generate(using: Int.arbitrary.suchThat { $0 > 0 }),
                // use the default boolean random generator
                displayFullProfile: c.generate()
            )
        }
    }
}
{% endhighlight %}

We now have a random `ViewModel` generator.

If you want to print a random instance of a `ViewModel`, you can do:

{% highlight swift %}
print(ViewModel.arbitrary.generate)

// Example of result (will change every time)
ViewModel(userName: "ybqe", messagesCount: 5, displayFullProfile: false)
{% endhighlight %}

Now testing our layout is straightforward:

{% highlight swift %}
property("Layout") <- forAll { (viewModel: ViewModel) in
    let view = MyView()
    view.configure(with: viewModel)
    view.setNeedsLayout()
    view.layoutIfNeeded()
    return view.hasNoAutoLayoutIssues && view.hasNoFrameOverlap
}

{% endhighlight %}

The implementation of the methods `hasNoAutoLayoutIssues` and `hasNoFrameOverlap` is left as an exercise to the reader, but some time ago, LinkedIn created a [library](https://github.com/linkedin/LayoutTest-iOS) heavily inspired by this approach (even so they do not provide real random values). You can find the implementations on the repository.

# Conclusion

We have seen how powerful property based testing is. Instead of writing one or two example based tests per feature, it allows you to generate thousands of random tests very easily.

The drawback is that this is not always straightforward to find a property that holds for every test you want to write. So even if you can't use this technique right now, keep it in mind the next time you need it.
