---
layout: post
title:  "DotSwift, we were there"
author: "Pierre Felgines"
---

This article was first published on Fabernovel [blog](https://en.fabernovel.com/insights/tech-en/dotswift-we-were-there).

On Friday, January 27th was the dotSwift event, a set of conferences for Swift developers in Paris. Three of our Applidium iOS engineers attended and here are their feedbacks.

# The event

The event took place in the Théâtre des Variétés, in the center of Paris. Good iOS related conferences are not used to be located in Paris so we were delighted to enjoy this event in such a peculiar location.

During the afternoon, there were three one-hour sessions, followed by half an hour breaks to network with other attendees or to enjoy the food and coffee.

The middle session was more focused on lightning talks, i.e. small talks of about 5 to 10 minutes. This second session was really dynamic as more speakers could come on stage. Lightning talks are a great opportunity for developers to present one particular subject, that would not cover an entire session.

The speakers had different backgrounds: there were engineers from well-known companies such as Realm, IBM or Microsoft, others were independent developers and we have also seen some involved in the Swift open source project. Daniel Steinberg was the MC, in charge of introducing the talks and asking questions to the speakers afterwards.

# The content

Talks were not all of the same level. Some were for Swift beginners, others for more experienced developers. But this is a good thing: everyone should learn something by attending the event.

Videos of the conferences won’t be available until a few weeks, so we won’t go into the details of each of them. We’ll present a quick summary of the different subjects.

Most of the conferences were focused on the Swift language itself and how to leverage Swift features in different use cases. For example, how to use the power of [structs](https://speakerdeck.com/dimdl/subclassing-structs-dotswift-1), how to make a good use of Protocol Oriented Programming, and so on.

A few talks were about the interoperability between Swift and Objective-C. How to avoid the propagation of `NSObject` or how to avoid the dynamic methods of Objective-C.

One talk was focused on [RxSwift](https://speakerdeck.com/icanzilb/rxswift-on-ios). This is a trendy topic and Marin Todorov did a great job explaining how we could use it in our codebase.

Last but not least, a couple of talks were focused on the server side of Swift. Ian Partridge, IBM engineer, was there to [talk](https://speakerdeck.com/ianpartridge/beyond-mobile) about it. Swift can run on Linux, but there are some pain points: for example some features of Foundation are not implemented yet.

# What we thought

Attending the dotSwift, we found out that traditional and simple subjects about Swift itself are not treated anymore. Instead, the talks presented some edge cases from real life projects (API design, [Bluetooth communication](https://speakerdeck.com/huguesbr/iot-and-ios-lessons-learned), etc), or new usages and new frameworks (RxSwift, swift on the server).

We think this testifies to a maturity both in the language and the community.

Since its open source publication, Swift has been growing quickly, and the community is really involved. Everyone can choose to participate in the next steps of the language and we think that is encouraging and a guarantee the project will head in the right direction.

What is even more encouraging is the will from [Apple](https://swift.org/server-apis/) and the community to port Swift to the server. Frameworks about server-side usage of Swift are now numerous: [Vapor](https://github.com/vapor/vapor) (Logan Wright, the co-creator was one of the speakers), [Kitura](https://github.com/IBM-Swift/Kitura) (powered by IBM, Ian Partridge was also a speaker), [Zewo](https://github.com/Zewo/Zewo), [Perfect](https://github.com/PerfectlySoft/Perfect), etc.

At Applidium we can’t wait to see what server applications will be created in the future. Ian Partridge emitted the idea that ideally we could develop two backends, one in Swift that communicates with the iOS application, and one in JavaScript that communicates with the front-end web application. He argued that, in this configuration, there will be an impact on how teams are working, allowing the same developers to work on both platforms, or to completely reuse code.

We are skeptical about those arguments because there are some drawbacks as well. Replicating the exact same features in two different languages by two different developers can lead to code replication and other problems that come along:  more bugs, subtle differences between the two platforms, waste of time and human resources. In addition, even if the Swift team works both on backend and front end, sharing code may be more complicated than it seems, because architectures and data models may differ.

In any case, we will see in the future what will happen, but for now we think it is too early to consider using those new server-side frameworks in production.

# Silver bullets

Here is a special mention to the talk of Roy Marmelstein. It was really engaging and we all agreed with him.

The main idea was that every time a new topic is trending, we experiment with it, and tend to push it everywhere in our codebase because we think it’s the next big thing, and you know, it solves all our problems. But some bigger issues may arise, and it is sometimes necessary to take a step back, to refactor, or even to admit it was not a good decision and rollback.

He compares those hot topics to _silver bullets_, in reference to this weapon that could kill any monster.

He specifically mentioned three silver bullets:

- Protocol Oriented Programming
- RxSwift
- Swift itself

In essence Protocol Oriented Programming is really powerful and Apple uses it all over the standard library. There was even an [entire session](https://developer.apple.com/videos/play/wwdc2015/408/) dedicated to it in the 2015 WWDC. But sometimes it can also be overused in situations where it does not really make sense.

RxSwift is attractive and simplifies the data flow, but the learning curve is pretty steep and it takes time to have a whole team able to use it effectively.

Swift itself can be seen as a _silver bullet_ as well, because the language is still in early development and suffers from ABI compatibility or long compilation times. There are cases where Objective-C is still the right choice.
What we can learn from this talk, and from the event itself is that all these new tools that were presented to us are really powerful and can really help improve our code. But each one is useful in a particular context and comes with drawbacks that needs to be acknowledged and understood before using it everywhere.

# Conclusion

We all agree the event was a success and the conferences were interesting. At Applidium we use Swift daily in our projects, so we did not learn anything groundbreaking, but it is always nice to come back to the office the next Monday with new ideas and new patterns to try.

What’s more, it is quite exciting to meet speakers in real life when you are used to reading their blogs.
