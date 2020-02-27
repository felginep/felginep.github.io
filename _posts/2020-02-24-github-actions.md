---
layout: post
title: "Github Actions"
author: "Pierre Felgines"
---

I recently wrote a public pod to share some common classes in my team. Since [Github Actions](https://github.com/features/actions){:target="_blank"} came out I never had the chance to test them. So that was the perfect time to give it a shot.

I use them to validate that my library is compiling, that all the tests pass, that the codebase respects our coding style and that the podspec is valid.

Let's see what are the simple steps you need to do.

# Creating the pod

To create a new pod you just have to run `pod lib create MyAwesomePod`. This will bootstrap a new project, with directories and placeholder classes.

Once it's done, the first thing to do is to create a `Gemfile` in the `Example` folder to lock the versions of your ruby dependencies.

{% highlight ruby %}
source 'https://rubygems.org'

gem 'fastlane', '<3.0'
gem 'cocoapods', '1.8.4'
gem 'CFPropertyList', '3.0.0'
{% endhighlight %}

You also want to create a `Fastfile` in the directory `Example/fastlane` and add a lane to compile and run the unit tests.

{% highlight ruby %}
desc "Run all unit tests"
lane :tests do
  scan(
    workspace: "MyAwesomePod.xcworkspace",
    configuration: "Debug",
    scheme: "MyAwesomePod-Example",
    clean: true,
    devices: ["iPhone 8"]
  )
end
{% endhighlight %}

Now, you can run your unit tests with a simple command: `bundle exec fastlane tests`.

# Linting source files

If you plan to allow other people to maintain your codebase, you may want to enforce some coding style rules. This can be easily done with [Swiftlint](https://github.com/realm/SwiftLint){:target="_blank"}.

First add the dependency to the `Example/Podfile` like so:

{% highlight ruby %}
use_frameworks!

target 'MyAwesomePod_Example' do
  pod 'MyAwesomePod', :path => '../'
  pod 'SwiftLint'

  target 'MyAwesomePod_Tests' do
    inherit! :search_paths
  end
end
{% endhighlight %}

Then create a `.swiftlint.yml` file in the `Example` folder to list all the rules you want to apply. The source files directories are
- `MyAwesomePod` folder for the pod source files
- `Example/MyAwesomePod` folder for the example source files

{% highlight yaml %}
disabled_rules:
  ...
opt_in_rules:
  ...
excluded:
  - Carthage
  - Pods
included:
  - ../MyAwesomePod
  - MyAwesomePod
{% endhighlight %}

Finally, add a fastlane action to your `Fastfile` to run `swiftlint`:

{% highlight ruby %}
desc "Linting"
lane :lint do
  swiftlint(
      mode: :lint,
      config_file: ".swiftlint.yml",
      strict: true,
      executable: "Pods/SwiftLint/swiftlint"
  )
end
{% endhighlight %}

# Github Actions

Now that we laid the groundwork, we can add the Github workflow.

Create a `.github/workflows/main.yml` file with this content:

{% highlight yaml %}
name: CI

on: [push]

jobs:
  build:

    runs-on: macOS-latest

    steps:
    - uses: actions/checkout@v2
    - name: Bundle install
      working-directory: ./Example
      run: bundle install
    - name: Pod install
      working-directory: ./Example
      run: bundle exec pod install
    - name: Build and test
      working-directory: ./Example
      run: bundle exec fastlane tests
    - name: Swiftlint
      working-directory: ./Example
      run: bundle exec fastlane lint
    - name: Pod lib lint
      run: pod lib lint
{% endhighlight %}

Each time you push on the repository, the workflow will:
- install ruby dependencies
- install pod dependencies
- compile the project and run the tests
- check the coding style
- validate the pod with `pod lib lint`

# Conclusion

We saw how easy it was to set up Github Actions with fastlane. With this simple workflow you can be confident that all those checks will run every time you or other contributors push to the repository.

If you are interested in all the actions available, you can find them [here](https://github.com/marketplace?type=actions).

