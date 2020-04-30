---
layout: post
title: Xcode at scale - Final conclusions
author: juli
---

This is the last step of a serie named **iOS at Scale** based on the next steps:
- [Introduction]({% link _posts/2020/2020-04-18-ios-at-scale-step0-introduction.md %})
- [Xcode at Scale]({% link _posts/2020/2020-04-18-ios-at-scale-step1-remove-xcode-conflicts.md %})
- [Refactor MarvelClient - split client from logic]({% link _posts/2020/2020-04-18-ios-at-scale-step2-refactor-marvel-client.md %})
- [Refactor DataProvidersKit - applying Iversion of Control]({% link _posts/2020/2020-04-18-ios-at-scale-step3-refactor-data-providers-kit.md %})
- [Refactor DetailKit - Single data flow, states, type erasure and more]({% link _posts/2020/2020-04-18-ios-at-scale-step4-single-data-flow.md %})
- [Refactor Navigator - Back to simplest]({% link _posts/2020/2020-04-18-ios-at-scale-step5-refactor-navigator-kit.md %})
- [Refactor AppCore - we really need it?]({% link _posts/2020/2020-04-18-ios-at-scale-step6-app-core-module.md %})
- [Dependency Injection - flexible and composable]({% link _posts/2020/2020-04-18-ios-at-scale-step7-dependency-injection.md %})
- [Extra ball: Cache made easy]({% link _posts/2020/2020-04-18-ios-at-scale-step8-cache-made-easy.md %})
- [Extra ball: Introducing to Promises 1]({% link _posts/2020/2020-04-18-ios-at-scale-step9-introduce-promises.md %})
- [Extra ball: Introducing to Promises 2]({% link _posts/2020/2020-04-18-ios-at-scale-step10-refactor-promises.md %})
- [Final conclusions]({% link _posts/2020/2020-04-18-ios-at-scale-step11-conclusions.md %})

_This is the end, my only friend, the end._

Let's do a little recap over what we've done here.

- How remove Xcode conclicts and support Xcode on Scale, having configurations in files instead the project, allowing us to have these configurations in a format understandable on our versioning system.
- How with `Tuist` we can make our life easier, better and more productive creating modules.
- How we can create SPM Packages without pain to have the option of reuse code along our whole company, or our projects.
- How use apple guidelines to give our protocols better names.
- How reduce code using maps over `Result` and the same concept on the Promises, and how this could make our code easier to understand, faster to develop, reduce the quantity and make harder to introduce bugs.
- How we can develop in a few steps our own Networking library.
- How we can develop our custom Storage library, and how we can integrate cache fast and easy.
- How we can develop our Promises framwork.
- How to extend a SPM package with more targets to extend functionality without touching the original code.
- How we can perform a refactor on a big project lead by little steps and keep our application working to be able to build new features in the meanwhile.
- We also noticed about some bad practices that have been removed.
- We created small composable objects, that are rusable and extensibles, applying POP.
- How we can have reactive views with Single Data Flow applying new techniques like Type Erasure. Keeping decoupled our Views from the ViewModels.
- We saw how a good component like the Navigator that we read from one post, can be not so useful if we don't understand fine how and why it works.
- How priorize functions and types over Classes, like the HTTPRequests and the new endpoint creation. And how this can minify our code.
- How apply patterns like IoC without Pain and make the Core of our app works decoupled from the MarvelClient.
- How do dependency injection in our app, without have our modules dependents about how we do it.


This serie is designed, to show and cover how maintain a big project, how we can apply new techniques and tools to our daily tasks and be more productive, while we can reduce our code quantity at the same time that we get more code quality, our software is more solid, and with more ability to change over the years.

I don't talked anything about [SwiftLint](https://github.com/realm/SwiftLint) or [SwiftFormat](https://github.com/nicklockwood/SwiftFormat), but they help a lot to maintain the same code style, to fail first and of course, to avoid things like `functionName(){` instead of `functionName() {`. This tools are very easy to add to our projects, and really help indexing, formatting and maintaining our code. You may find a project with those tools and tuist [here](https://github.com/JulianAlonso/MarvelApp).

Working in large teams, with many people, we have to limit very well ourselves to be ables to complete tasks without the needs of touching a lot of code in colindant modules, make simple functions, and create components easy to extend and share, are a must. Nowadays we have tools like SPM or Tuist that help us defining those Modules, encapsulate them, and give us the ability to share and reuse like we need.

The main difference when working on a true professional project, I guess that it doesn't reside all on the code, the code have a great percentage, you know, but, we forget about the project setting up. We must automate as many as we can our configurations, provide a [Makefile](https://opensource.com/article/18/8/what-how-makefile) (or whatever scripting tool) to help if one person is new on our company, it could run `make project` and has the repostiory working, with dependencies downloaded, and the project automatically generated. That must be our goal. But not only make the new mates an easier onboarding, this way our project will be easy reproducible on our CI system, and if someone need to know how the project it's beign built, only looking to one file can understand all the steps, because they are defined.

Finally, I hope you enjoyed the journey as many as I've enjoyed developing it. Our code vision must change, we must priorize to use this kind of new techniques over the old ones. Apple is going for that, SwiftUI and Combine are a fact. But not only apple is leading us for those techniques, Android has Live Data, ViewModels, Eithers, are supported from Google, one more time, leading us to build reactive and more functional apps. Because the code reduction is incredible. How our code decrease along it still doing the same or more, for me has no words. All of this, without forget protocol oriented programming, how extensions works, and how combine all of these techniques build better and solid software.

You can find me on [Twitter](https://twitter.com/MaisterJuli)

## Steps

- [Introduction]({% link _posts/2020/2020-04-18-ios-at-scale-step0-introduction.md %})
- [Xcode at Scale]({% link _posts/2020/2020-04-18-ios-at-scale-step1-remove-xcode-conflicts.md %})
- [Refactor MarvelClient - split client from logic]({% link _posts/2020/2020-04-18-ios-at-scale-step2-refactor-marvel-client.md %})
- [Refactor DataProvidersKit - applying Iversion of Control]({% link _posts/2020/2020-04-18-ios-at-scale-step3-refactor-data-providers-kit.md %})
- [Refactor DetailKit - Single data flow, states, type erasure and more]({% link _posts/2020/2020-04-18-ios-at-scale-step4-single-data-flow.md %})
- [Refactor Navigator - Back to simplest]({% link _posts/2020/2020-04-18-ios-at-scale-step5-refactor-navigator-kit.md %})
- [Refactor AppCore - we really need it?]({% link _posts/2020/2020-04-18-ios-at-scale-step6-app-core-module.md %})
- [Dependency Injection - flexible and composable]({% link _posts/2020/2020-04-18-ios-at-scale-step7-dependency-injection.md %})
- [Extra ball: Cache made easy]({% link _posts/2020/2020-04-18-ios-at-scale-step8-cache-made-easy.md %})
- [Extra ball: Introducing to Promises 1]({% link _posts/2020/2020-04-18-ios-at-scale-step9-introduce-promises.md %})
- [Extra ball: Introducing to Promises 2]({% link _posts/2020/2020-04-18-ios-at-scale-step10-refactor-promises.md %})
- [Final conclusions]({% link _posts/2020/2020-04-18-ios-at-scale-step11-conclusions.md %})