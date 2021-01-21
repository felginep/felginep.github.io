---
layout: post
title: "Visit Monitoring"
author: "Pierre Felgines"
---

As Covid-19 started to spread, we tried to find a way to help as developers, with the tools at our disposal. One idea was to use out of the box [Visit Monitoring](https://developer.apple.com/documentation/corelocation/getting_the_user_s_location/using_the_visits_location_service) service proposed by Apple to match locations with infected people. The idea at the time was the same as the [ExposureNotification](https://developer.apple.com/documentation/exposurenotification) framework, but using location service instead of bluetooth.

That's a good opportunity to focus on visit monitoring here and see if it's a good candidate for this use case.

## The service

### Setup

Visit monitoring requires the `Always` authorization for location. The service is not available when `When In Use` is requested. You can find all the location services and their needed authorizations [here](https://developer.apple.com/documentation/corelocation/choosing_the_location_services_authorization_to_request#3376460).

Visit monitoring is meant to be used in background. Like the other background services, it will continue to run even when the app is suspended, and will [launch](https://developer.apple.com/documentation/corelocation/getting_the_user_s_location/handling_location_events_in_the_background#2865362) the app in the background to deliver events.
Make sure to enable `Location Updates` in the application background modes capabilities and to set `allowsBackgroundLocationUpdates` to `true` for the location manager.

### Start monitoring visits

The use is really straightforward, as for other location services.

First create your `CLLocationManager` and call the method `startMonitoringVisits()`. If the service is authorized and running, you will receive callbacks on the delegate method `locationManager(_:didVisit:)`.

Here is a skeleton of implementation of such a feature.

{% highlight swift %}
import CoreLocation

class LocationService: NSObject, CLLocationManagerDelegate {

    private let locationManager: CLLocationManager
    private var pendingAuthorizationCompletion: ((Bool) -> Void)?

    init(locationManager: CLLocationManager = CLLocationManager()) {
        self.locationManager = locationManager
        super.init()
        locationManager.allowsBackgroundLocationUpdates = true
        locationManager.delegate = self
    }

    // MARK: - LocationService

    func start() {
        switch CLLocationManager.authorizationStatus() {
        case .notDetermined:
            locationManager.requestAlwaysAuthorization()
        case .authorizedAlways,
             .authorizedWhenInUse,
             .restricted,
             .denied:
            // Handle unauthorized
        @unknown default:
            break
        }
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager,
                         didChangeAuthorization status: CLAuthorizationStatus) {
        switch status {
        case .authorizedAlways:
            locationManager.startMonitoringVisits()
        case .notDetermined,
             .authorizedWhenInUse,
             .restricted,
             .denied:
             // Handle unauthorized
        @unknown default:
            break
        }
    }

    func locationManager(_ manager: CLLocationManager,
                         didVisit visit: CLVisit) {
        // Handle visit
    }
}
{% endhighlight %}

### Debuging

The service runs in background and you have no idea when the system will choose to trigger a visit update. That's why it's difficult to debug it.

Something you can do to help debug the application is to trigger a local notification on the device every time the device logs a new visit. That way, when using your device in your daily life, you'll be able to know precisely when and where the visits occur.

{% highlight swift %}
func locationManager(_ manager: CLLocationManager, didVisit visit: CLVisit) {
    postDebugMessage(for: visit)
    // ...
}

private func postDebugMessage(for visit: CLVisit) {
    let content = UNMutableNotificationContent()
    content.title = "New place registered"
    content.body = "\(visit.coordinate)"
    content.sound = .default
    let trigger = UNTimeIntervalNotificationTrigger(
        timeInterval: 1,
        repeats: false
    )
    let request = UNNotificationRequest(
        identifier: "",
        content: content,
        trigger: trigger
    )
    center.add(request, withCompletionHandler: nil)
}
{% endhighlight %}

## Pros and cons

### Pros

- The major feature of visit monitoring is to be able to receive user locations with a very minimal impact on battery life. Apple even describes this service as _the most power-efficient way of gathering location data_. The goal here is not to be very accurate about the user trace, but to be able to retrieve context from a user location.

- Apple uses sophisticated algorithms to monitor for places that the user might consider a noteworthy part of their day. For instance the property `pausesLocationUpdatesAutomatically` automatically pauses the service if the user is unlikely to move. That means the framework learns from your habits and trigger events accordingly.

### Cons

- Some visits returned by the system are incomplete and the documentation is misleading. The documentation states that _Visit objects contain as much information about the visit as possible but may not always include both the arrival and departure times_.

In Swift, though, the visit class contains non optional `arrivalDate` and `departureDate`, as we could have expected. These two properties rather contain _invalid_ data (like `distantPast` or `distantFuture`):

{% highlight swift %}
@available(iOS 8.0, *)
class CLVisit : NSObject, NSSecureCoding, NSCopying {
    // This may be equal to [NSDate distantPast]
    // if the true arrival date isn't available.
    var arrivalDate: Date { get }
    // This is equal to [NSDate distantFuture]
    // if the device hasn't yet left.
    var departureDate: Date { get }
    var coordinate: CLLocationCoordinate2D { get }
    var horizontalAccuracy: CLLocationAccuracy { get }
}
{% endhighlight %}

That means you can't trust the visit dates and have to discard invalid visits if you rely on both arrival and departure dates.

- What's more, the precision of the visit, described in the property `horizontalAccuracy` in meters, is not very precise. The center of the visit (described as the `coordinate` property) may not be the exact location where you were during the visit. That means you should not rely on the `coordinate` property to  track the precise location of the user.

- Another issue about precision is that you can't know the elevation of the user. That means you may have a single event when the user visits a building, although he may have been in very different levels for different purposes.

## Conclusion

Visit monitoring is a powerful service that should be considered as an alternative to Significant-change monitoring in some cases. Its minimal impact on battery life makes it perfect to use in situations where you just need to get context from a noteworthy place: restaurant, gym, home, work, etc.

In our use case we wanted to match locations of multiple users during the same period of time. But as we have seen previously, arrival and departure dates are not always present and the location is not precise. So visit monitoring is not well suited for this situation.

_Note: This article was also published on Fabernovel [blog](https://www.fabernovel.com/en/engineering/visit-monitoring-ios)_.

