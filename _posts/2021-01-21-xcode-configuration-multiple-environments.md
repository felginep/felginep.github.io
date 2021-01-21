---
layout: post
title: "Xcode configuration for multiple environments"
author: "Pierre Felgines"
---

I recently stumbled upon yet another article that explains different techniques to configure an Xcode project for multiple environments. Most of the articles out there propose the same solutions and although these are valid techniques, I think Xcode is not giving us the right tools and that there is room for improvement.

Let's see what the proposed solutions are, the issues associated with them and how we can do better.

## The (not so) recommended setup

The root of the problem is to be able to handle multiple development environments for an Xcode project. For each environment we want to define dynamic environment variables, such as server url, logging, feature flagging, etc. And each environment should be compiled in two modes: `Debug` (for local development, or to run tests) and `Release` (to distribute on the stores).

We can sum up the requirements we want to achieve with the following matrix:


| Build \ Environment       | Development         | Production         | ... |
|---------|---------------------|--------------------|--------------------|
| Debug   | Development Debug   | Production Debug   | ... Debug   |
| Release | Development Release | Production Release | ... Release |

Many solutions out there propose to get rid of the default `Debug` and `Release` build configurations and to instead create two build configurations for each environment.

For instance if we have a Development and a Production environment, we will create four configurations (all the possibilities from the matrix above):
- Development Debug
- Development Release
- Production Debug
- Production Release

Once we have these build configurations, we need to store the environment variables for each environment. The majority of resources I found propose one of the following:
- to use `.xcconfig` files for each build configuration (with the issue of [escaping characters](https://stackoverflow.com/q/21317844))
- to store values directly in Xcode build settings for each configuration (with User Defined settings)

## The drawbacks

There are a few drawbacks with the previous approach.

The main issue is that we correlate compilation options defined in the project Build Settings and development environments. These are two separate things that should not be treated the same way. The issue here is mostly due to Xcode that only provides Build Configurations to store the build settings and Schemes to choose the configuration used during the build.

In practice, for each new environment we want to add, we have to create a `Debug` and a `Release` configuration. That's not a very scalable solution when we have a lot of development environments.

What's more, the second issue is that we will find ourselves duplicating a lot of build settings between all these configurations. For instance, both DevelopmentDebug and ProductionDebug will share the same build settings associated with compilation. There is no reason these settings should diverge between Development and Production.

## Another approach

We saw earlier that it's a bad practice to mix compilation build settings and development environment.

I propose to stick with only two configurations, `Debug` and `Release`, that come for free when creating a new Xcode project. All compilation options will be set in the build settings.

To handle the environment variables that change for each environment, we will store them in yaml files on disk.

For instance, here is the content of the `env.development.yml`:

{% highlight yml %}
pbxproj:
  PRODUCT_BUNDLE_IDENTIFIER: "com.felginep.DemoBlog.dev"
env:
  baseUrl: "dev.demoblog.api.com/" # the end point url
  logLevel: "debug" # the log level used by the logger in the app
  # ... any other variable ...
{% endhighlight %}

We can notice two things here:
- the values under the `pbxproj` key represent the values that will be integrated to the build settings of Xcode and that are related to the environment and not to the compilation
- the values under the `env` key represent the environment variables that will be used at runtime

Note: the main advantage to use yaml to store our environment variables is that we can add comments alongside the values.

The next step is to create a [script](https://github.com/felginep/DemoBlog/blob/main/prepare.rb) that will read the yaml files and integrate their values in the project. Any scripting language will do, but I personally prefer Ruby because it's easy to handle yaml and to integrate with fastlane.

So let's say we have a script that we can call like this:

{% highlight sh %}
./prepare.rb development
{% endhighlight %}

The script will read the `env.development.yml` and do two things:
- integrate the environment build settings values (under the `pbxproj` key) into Xcode
- generate a Swift file that represents our environment variables (under the `env` key)

### Updating the pbxproj

To integrate the environment build settings into our project, the script will copy the values from the yaml to an xcconfig file named `Environment.xcconfig`. This file is auto generated so we can add it to the `.gitignore`.

In our example, the file `Environment.xcconfig` that is generated would look like this:

{% highlight text %}
// Environment.xcconfig

PRODUCT_BUNDLE_IDENTIFIER = com.felginep.DemoBlog.dev
{% endhighlight %}

Once this `Environment.xcconfig` is created, we need to tell Xcode to take it into account. For this, we create two more xcconfig files, `Debug.xcconfig` and `Release.xcconfig`, one for each configuration.

Their content is the following:

{% highlight text %}
// DemoAppDebug.xcconfig

#include "Environment.xcconfig"
#include "../../Pods/Target Support Files/Pods-DemoBlog/Pods-DemoBlog.debug.xcconfig"

// Here add any build settings you want for Debug configuration
{% endhighlight %}

{% highlight text %}
// DemoAppRelease.xcconfig

#include "Environment.xcconfig"
#include "../../Pods/Target Support Files/Pods-DemoBlog/Pods-DemoBlog.release.xcconfig"

// Here add any build settings you want for Release configuration
{% endhighlight %}

Note: we include the cocoapods generated xcconfigs as they are not automatically loaded by cocoapods anymore. If we forget to include them, we will see a warning during `pod install`.

The last step is to specify in Xcode that we want to use these new xcconfig files for each of our build configurations.

{% include image.html
    img="assets/xcode-configuration/xcode-configuration.png"
    title="Integration of xcconfig files"
    caption="Integration of xcconfig files" %}

Finally, we can verify that the xcconfig has been correctly loaded because the bundle identifier has been updated for the environment:

{% include image.html
    img="assets/xcode-configuration/product-bundle-identifier.png"
    title="Bundle identifier for development"
    caption="Bundle identifier for development" %}

To recap what we just did here:
- we copy all the values from the `pbxproj` key of our yaml into a `Environment.xcconfig` file (which is in the `.gitignore`)
- this `Environment.xcconfig` is then loaded for each configuration in the files `Debug.xcconfig` and `Release.xcconfig`
- the files `Debug.xcconfig` and `Release.xcconfig` are associated in Xcode for each configuration `Debug` and `Release`

If we run the script for another environment, only the values contained in `Environment.xcconfig` will change, and they will be automatically loaded by Xcode.

### Environment variables

The final step of our solution is to handle the environment variables that will be used at runtime. The idea here is to define an `Environment` struct that will hold the values from our yaml:

{% highlight swift %}
// Environment.swift

struct Environment {
    let baseUrl: String
    let logLevel: String
}
{% endhighlight %}

As we did with the `Environment.xcconfig`, the script will generate a file `Environment+Current.swift` that contains all the values from the `env` key of the yaml. This file will also be added in the `.gitignore` because it will change each time we change the environment.

In our example, for the development environment, the script will create the following file:

{% highlight swift %}
// Environment+Current.swift

extension Environment {

    static let current = Environment(
         baseUrl: "dev.demoblog.api.com/",
         logLevel: "debug"
    )
}
{% endhighlight %}

In the code, we now can use the `Enviroment.current` value wherever we want. We can also create our own instance of `Environment` for test purposes, or for different sections of the app.

The real power here is that the environment is a first class citizen in our app and values are verified at **compile time**. If we forget to add a key in a yaml file, or if the key has the wrong type, we will get an error during compilation.

## In practice

So how does it work in practice?

Anytime we want to change the **development environment**, we run the following **script** command: `./prepare.rb [development|production]`. This will generate two files `Environment.xcconfig` and `Environment+Current.swift` that are in the `.gitignore`. These files contain the values defined in the yaml. `Environment.xcconfig` will contain all the build settings that can change with the environment, whereas `Environment+Current.swift` will contain all the dynamic environment variables and will be verified at compile time.

When we want to change the **build configuration** (meaning the compilation options), we change the **scheme** to choose between `Debug` and `Release`. We now have a lot more flexibility to choose which environment to build in which configuration. What's more, when we add a new environment in our project, all we need to do is to create a new yaml file with the correct values, run the script with this new environment, and that's it. There is no need to create new build configurations anymore, nor to think about which compilation options will be associated.

You can find the demo project I used in this article [here](https://github.com/felginep/DemoBlog). Note that the [script](https://github.com/felginep/DemoBlog/blob/main/prepare.rb) is just for demonstration purposes and feel free to improve it for your specific needs. For instance, you may use it to load Firebase plist instead of relying of the build phases.
