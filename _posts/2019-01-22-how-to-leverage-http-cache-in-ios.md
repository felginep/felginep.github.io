---
layout: post
title: "How to leverage HTTP cache in iOS"
author: "Pierre Felgines"
---

This article was first published on Fabernovel [blog](https://en.fabernovel.com/engineering/how-to-leverage-http-cache-in-ios).

One of the most difficult subject in programming is caching because there is no silver bullet to the problem and each solution comes with compromises.

In this article I want to focus on how we achieved basic caching in most of our screens by leveraging HTTP cache. The goal was to provide content to the user even if there is no internet connection, in the easiest way possible. The idea is rather simple: for all `GET` requests, we cache the response that we get. Then, if there is no connection, we fetch the previous response from the cache and display a warning message to the user, informing the data may be outdated.

## Caching the data

The core idea is to cache all the responses we receive.
This can be done with the method `urlSession(_:dataTask:willCacheResponse:completionHandler:)` of `URLSessionDataDelegate`.

The documentation of this method states:

> This method is called only if the NSURLProtocol handling the request decides to cache the response. As a rule, responses are cached only when all of the following are true:
- The request is for an HTTP or HTTPS URL (or your own custom networking protocol that supports caching).
- The request was successful (with a status code in the 200–299 range).
- The provided response came from the server, rather than out of the cache.
- The session configuration’s cache policy allows caching.
- The provided URLRequest object's cache policy (if applicable) allows caching.
- The cache-related headers in the server’s response (if present) allow caching.
- The response size is small enough to reasonably fit within the cache. (For example, if you provide a disk cache, the response must be no larger than about 5% of the disk cache size.)

Caching all the requests requires that the server returns the following headers: either an `Expires:` header or a `Cache-Control:` header with a `max-age` or `s-maxage` parameter.

Using Alamofire, here is what the code looks like:

{% highlight swift %}
dataTaskWillCacheResponse = { [weak self] (urlSession, urlSessionDataTask, cachedURLResponse) -> CachedURLResponse? in
    // Clean the response if needed
    let modifiedResponse = self?
        .whipeAuthenticationHeaders(from: cachedURLResponse.response)
        ?? cachedURLResponse.response

    // Store the current date with the cached response
    // (the date will be displayed later)
    let date = NSDate()
    let userInfo = [
    	Keys.cacheDateHeader: date.timeIntervalSince1970
    ]

    // Return a new CachedURLResponse with the userInfo above
    return CachedURLResponse(
        response: modifiedResponse,
        data: cachedURLResponse.data,
        userInfo: userInfo,
        storagePolicy: .allowed
    )
}
{% endhighlight %}

Note that we save the current date along with the response. We will use it later to display the freshness of the data.

You may wonder what the `whipeAuthenticationHeaders` method is doing here. If we save the response as is, when we get back the cached response from the local cache, the potential authentication headers would be outdated and would cause the next requests to fail because the token is expired. That’s why we remove all authentication headers before caching the response. That way they won’t be used by the application. This is ad hoc for our authentication method, but just be aware that kind of problem can happen.

## Fetching the cached data

The mechanism to fetch the cached response is contained in a *behavior*. A behavior is an object that can execute code at various times of a request life. It’s not the topic of this article, but you can learn more [here](http://khanlou.com/2017/01/request-behaviors/).
Just remember that in our case we want to modify the response when the network call fails, and return the cached data instead of an error.

{% highlight swift %}
class LoadFromCacheIfUnreachableBehavior: RequestBehavior {

    private let cache: URLCache

    init(cache: URLCache) {
        self.cache = cache
    }

    func modify(request: HTTPRequest, response: JSONResponse) -> JSONResponse {
    	// Verify that we need to use the cache
        guard case let .error(error) = response.result,
            let apiError = error as? APIError,
            apiError == .unreachableService else {
                return response
        }

        // Get the request
        guard let urlRequest = response.request else {
            return response
        }

        // Get the response
        guard let cachedResponse = cache.cachedResponse(for: urlRequest),
            let httpURLResponse = cachedResponse.response as? HTTPURLResponse else {
                return response
        }

        // Get the date we saved earlier back from the cache
        var date: Date?
        if let userInfo = cachedResponse.userInfo,
            let cacheDate = userInfo[Keys.cacheDateHeader] as? TimeInterval {
            date = Date(timeIntervalSince1970: cacheDate)
        }

        do {
            // Create the response from cached data
            let json = try JSON(data: cachedResponse.data)
            let newResponse = JSONResponse(
                result: .value(json),
                request: response.request,
                response: httpURLResponse,
                date: date
            )
            return newResponse
        } catch {
            return response
        }
    }
}
{% endhighlight %}

In simple terms, when we hit the network and get an error due to unreachable service (meaning an error from `URLSession`), we then ask the cached response for the current url request and return it.

## The display

In the rest of the app, the objects responsible to fetch data, called *repositories*, have method signatures like this one:

{% highlight swift %}
protocol AccountRepository {
    func getAccounts(completion: ((Result<CachedValue<[Account]>>) -> Void)?)
}
{% endhighlight %}

They use a `CachedValue` object defined as:

{% highlight swift %}
enum CachedValue<T> {
    case fresh(value: T)
    case cached(value: T, date: Date)
}
{% endhighlight %}

Either the data is fresh when the network call succeeds, either the data is read from the cache and contains the date of the cached response.

In the view controller, we display an information bar if the data is cached to inform the end user that an error occurred and that this is not the most recent data.

{% highlight swift %}
private func display(cachedAccounts: CachedValue<[Account]>) {
    self.accounts = cachedAccounts.value
    tableView.reloadData()

    if let date = cachedAccounts.date {
        displayInformationBar(date: date)
    } else {
        hideInformationBar()
    }
}
{% endhighlight %}

When the network request fails, the result looks like this. An information bar is displayed to help the user understand what is going on.

{% include image.html
            img="assets/http-cache-information-bar.png"
            title="Image 1. Information bar displayed to the user"
            caption="Image 1. Information bar displayed to the user"
            max-width="432px" %}

## Conclusion

We have seen how to create a local cache system based only on HTTP cache. This implementation only works for `GET` requests, of course, and it’s very far from an offline experience, but the users will see some content even if they have no network connection. This is already a good start and a nice improvement compared to just displaying an error.










