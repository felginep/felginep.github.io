---
layout: post
title: "Using intents to leave the app"
author: "Pierre Felgines"
---

In an iOS application, we often have to redirect the user outside of the application for a variety of reasons: calling a number, sharing content, writing an email, etc.

UIKit comes with a number of built in ways to perform these actions but there is not an unified method to do it. And sometimes you even want to use third party schemes instead of native ones (using Google Maps instead of Maps for instance).

Let's see with a few examples how we could hide this logic into more abstract types and simplify the calling site.

# Mail

Let's say we want to display the user a mail compose sheet. Thanks to the documentation, the right class to use is `MFMailComposeViewController`.

> Use this view controller to display a standard email interface inside your app.

The detail here is that `MFMailComposeViewController` is a class of the `MessageUI` framework. And this detail bothers me because I try to keep the imported frameworks to the minimum. I don't want to import `MessageUI` in my view controller class, and rather hide implementation details.

A solution in this case is to use the `MFMailComposeViewController` in a specific object that will not be visible to the external world.

For this, let's borrow the *intent* naming of Android developers and create a `MailIntent` that will be used by the sender to inform its intent to display a mail interface, but without knowing how.

{% highlight swift %}
protocol MailIntent {
    func mail(to recipient: String)
}
{% endhighlight %}

The caller can use it like so:

{% highlight swift %}
class ContactViewController {

    private let mailIntent: MailIntent

    ...

    func requestEmail(to recipient: String) {
        mailIntent.mail(to: recipient)
    }
}
{% endhighlight %}


The implementation of `MailIntent` can import `MessageUI` and use `MFMailComposeViewController`. All the `MessageUI` code is constrained into this class. That also means we can use the intent in different places in the application, the display of the mail interface will always be the same and we won't have to repeat ourselves.

{% highlight swift %}
import MessageUI

class NativeMailIntent: NSObject,
    MailIntent,
    MFMailComposeViewControllerDelegate {

    private let viewController: UIViewController

    init(viewController: UIViewController) {
        self.viewController = viewController
    }

    // MARK: - MailIntent

    func mail(to recipient: String) {
        guard MFMailComposeViewController.canSendMail() else {
            return
        }
        let mailComposeViewController = MFMailComposeViewController()
        mailComposeViewController.mailComposeDelegate = self
        mailComposeViewController.setToRecipients([recipient])
        viewController.present(mailComposeViewController, animated: true)
    }

    // MARK: - MFMailComposeViewControllerDelegate

    func mailComposeController(_ controller: MFMailComposeViewController,
                               didFinishWith result: MFMailComposeResult,
                               error: Error?) {
        viewController.dismiss(animated: true)
    }
}
{% endhighlight %}

That also means that if we want to change the way we send an email, it's simple. For example if we choose to redirect the user to the Mail app instead of opening a compose sheet, we could just rewrite the `mail(to:)` function like so:

{% highlight swift %}

class NativeMailIntent: MailIntent {

    private let application: UIApplication

    init(application: UIApplication) {
        self.application = application
    }

    // MARK: - MailIntent

    func mail(to recipient: String) {
        let mailURLString = "mailto:\(mailTo.removeWhitespaces())"
        guard let url = URL(string: mailURLString),
            application.canOpenURL(url) else {
                return
        }
        application.open(url, options: [:])
    }
}
{% endhighlight %}

What's more, we all know that `MFMailComposeViewController` is not working on simulator. That means we can't be sure our actions that display the compose sheet are well implemented when we test on simulator (either manually or with UI tests). In this case we can create a dummy implementation of `MailIntent` that  will just display an alert controller with the recipient as the message.

{% highlight swift %}
class DebugMailIntent: MailIntent {

    private let viewController: UIViewController

    init(viewController: UIViewController) {
        self.viewController = viewController
    }

    // MARK: - MailIntent

    func mail(to recipient: String) {
        let alert = UIAlertController(
            title: "DebugMailIntent",
            message: "Recipient \(recipient)",
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "Ok", style: .cancel))
        viewController.present(alert, animated: true)
    }
}
{% endhighlight %}

# Map

We can apply the same technique for opening map items into Maps application and hide the import of the `MapKit` framework.


{% highlight swift %}
struct MapItem {
    let title: String
    let coordinate: CLLocationCoordinate2D
}

protocol MapIntent {
    func open(item: MapItem)
}
{% endhighlight %}

The implementation will be:

{% highlight swift %}
import MapKit

class NativeMapIntent: MapIntent {

    // MARK: - MapIntent

    func open(item: MapItem) {
        let placemark = MKPlacemark(
            coordinate: item.coordinate,
            addressDictionary: nil
        )
        let mapItem = MKMapItem(placemark: placemark)
        mapItem.name = item.title
        mapItem.openInMaps(launchOptions: [:])
    }
}
{% endhighlight %}

What's interesting in this case is that we can create another implementation for Google Maps, a third party application.

{% highlight swift %}
class GoogleMapIntent: MapIntent {

    private let application: UIApplication

    init(application: UIApplication) {
        self.application = application
    }

    // MARK: - MapIntent

    func open(item: MapItem) {
        let urlString = "comgooglemaps://?daddr=\(item.coordinate.latitude),\(item.coordinate.longitude)"
        guard let url = URL(string: urlString),
            application.canOpenURL(url) else {
                return
        }
        application.open(url, options: [:])
    }
}
{% endhighlight %}

# Phone Call

Even if there is no framework to hide in this case, the naming of the intent makes things very convenient to use.

{% highlight swift %}
protocol PhoneIntent {
    func call(phone: String)
}
{% endhighlight %}

The implementation will be:

{% highlight swift %}
class NativePhoneIntent: PhoneIntent {

    private let application: UIApplication

    init(application: UIApplication) {
        self.application = application
    }

    // MARK: - PhoneIntent

    func call(phone: String) {
        guard !phone.isEmpty else { return }
        let phoneURLString = "tel://\(phone.ad_removingWhitespaces())"
        guard let url = URL(string: phoneURLString),
            application.canOpenURL(url) else {
                return
        }
        application.open(url, options: [:])
    }
}
{% endhighlight %}

In this case we directly call the provided number, but we could imagine passing a view controller to the native intent, and display an alert before calling.

# Conclusion

This simple technique allows two things:
- hide the implementation details of the intent (import of frameworks) and keep things simple in the caller site
- easily reuse the same intent in different places

