---
layout: post
title: iOS at Scale, Step 4 - App core Module
author: juli
---

This is the seventh step of a serie named **iOS at Scale** based on the next steps:
- [Introduction]({% link _posts/2020/2020-04-18-ios-at-scale-step0-introduction.md %})
- [Xcode at Scale]({% link _posts/2020/2020-04-18-ios-at-scale-step1-remove-xcode-conflicts.md %})
- [Refactor MarvelClient - split client from logic]({% link _posts/2020/2020-04-18-ios-at-scale-step2-refactor-marvel-client.md %})
- [Refactor DataProvidersKit - applying Iversion of Control]({% link _posts/2020/2020-04-18-ios-at-scale-step3-refactor-data-providers-kit.md %})
- [Refactor DetailKit - Single data flow, states, type erasure and more]({% link _posts/2020/2020-04-18-ios-at-scale-step4-single-data-flow.md %})
- [Refactor Navigator - Back to simplest]({% link _posts/2020/2020-04-18-ios-at-scale-step5-refactor-navigator-kit.md %})
- [Refactor AppCore - we really need it?]({% link _posts/2020/2020-04-18-ios-at-scale-step6-app-core-module.md %})
- [Dependency Injection - flexible and composible]({% link _posts/2020/2020-04-18-ios-at-scale-step7-dependency-injection.md %})
- [Extra ball: Cache made easy]({% link _posts/2020/2020-04-18-ios-at-scale-step8-cache-made-easy.md %})
- [Extra ball: Introducing to Promises 1]({% link _posts/2020/2020-04-18-ios-at-scale-step9-introduce-promises.md %})
- [Extra ball: Introducing to Promises 2]({% link _posts/2020/2020-04-18-ios-at-scale-step10-refactor-promises.md %})
- [Final conclusions]({% link _posts/2020/2020-04-18-ios-at-scale-step11-conclusions.md %})

The final module has come. It's time to see what `AppCore` module it's giving us and why. Lets move to the step-5 on git.

One of the thoughts about this module, is that I don't understand **why** this module it's here. I'm fully agree about make our applications by modules, to limit those modules responsabilities, hide how that module works with access modifiers of our codes, and expose only the components that we need to expose. This is the most usual thing in other languages like `Java`. But before of create a Module, we must stop and think, We really need it? Becuase not always more it's better.

I guess the app target must have inside the `AppDelegate` and must be responsible of building the app. Talking about modularized apps, I have mixed feelings about _Feature Freamework_, about his opportunity cost, but I understand that on big teams, they limit very well what pieces of code has to touch each team, and that could and make up for the complexity it adds (note that with [Tuist](tuist.io) or other project generators this complixity it's very reduced). But, back to the `AppCore` module, that module it's doing what our app module has to do. Why we should have this module instead move the code the app directly?

AppCore it's an extra complexity layer that should be removed. Remember to keep all things as simple as possible.

## Refactor the app core module

The first thing to do, it's to move all the code inside `AppCore`, to the app target. (commit: `d30ebd9`)

Then, we must remove the `Module/AppCore` folder, and updates the dependencies of our app `Manifest`. (commit: `aa460e5`) 

With those simple steps, we keep our app compiling, and we've already removed an extra Module, that wasn't giving us anything.

Not let's going deeper inside the app target.

## Refactor the app target

One of the points that we should keep in mind when developping, it's to maintain the code as easy as possible. This is not an idea of mine, it's what [KISS principle](https://en.wikipedia.org/wiki/KISS_principle) says. I guess this principle is one of the most forgotten by us as developers, we always try to think on the future of the system what we're building, and some times we make hard things when it doesn't has reasons to be.

Let's see the `AppDelegate`. In this case, we here are doing exactly that, premature optimizations. We've built an `AppLaunchCoordinator`, to keep the code inside `AppDelegate` as less as possible, while all the methods on the AppLaunchCoordinator are with a comment, and should be removed.

Also notice that we don't gain anything removing all the code on our AppDelegate, but having all that code inside other class like the `AppLaunchCoordinator` has. This is only move the complexity of one class, to another, but not reduces it, backwards, it adds an extra complexity layer. So take this as a bad practice.

So, the first step to do it's remove the `AppLaunchCoordinator`. If inside our application, our `AppDelegate` is really simple, as this is, keep the code inside the app delegate. Then, if we discover that our app delegate's grows, I suggegst you to do one of the three exposed methods here [Refactor Massive AppDelegate](https://www.vadimbulavin.com/refactoring-massive-app-delegate/). Take the one that better fits on your needs.

Now, our app delegate looks like: (commit: `51c4bb9`)

It's time to fix the dependency graph. (commit: `29d2c8e`)

```swift
final class AppDelegate: UIResponder, UIApplicationDelegate {
    
    var window: UIWindow?
    var navigator: Navigator?

    public func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        setUpAppDelegateDependencies()
        setUpWindow()
        self.navigator = AppCoreKitAssembly.current.navigator
        navigator?.handle(navigation: .root(.list(assembly: AppCoreKitAssembly.current.charactersListKit)))
        return true
    }
    
}

private extension AppDelegate {
  
    func setUpAppDelegateDependencies() {
        if AppCoreKitAssembly.current == nil {
            AppCoreKitAssembly.current = AppCoreKitAssembly()
        }
    }
    
    func setUpWindow() {
        window = AppCoreKitAssembly.window
        window!.frame = UIScreen.main.bounds
    }
    
}
```

Now, we can make this even simpler.

Like swift [static lets are lazy](https://alisoftware.github.io/swift/2016/02/28/being-lazy/), let's make `AppCoreKitAssembly` being a swift singleton. Let's create a `private init` and then make the current instanciate himself. (commit: `42a7e3f`)

Then, we can remove the `setUpAppDelegateDependencies` on the `AppDelegate`. And we end up with an AppDelegate like this: (commit: `a5f47e6`)

```swift
final class AppDelegate: UIResponder, UIApplicationDelegate {
    
    var window: UIWindow?
    var navigator: Navigator?

    public func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        self.window = AppCoreKitAssembly.current.window
        self.navigator = AppCoreKitAssembly.current.navigator
        navigator?.handle(navigation: .root(.list(assembly: AppCoreKitAssembly.current.charactersListKit)))
        return true
    }
    
}
```

## Conclusions

This chapter has been short. But what we've done?
- We've removed the `AppCore` extra module that wasn't giving us nothing more complexity.
- We've removed extra complexity inside the `AppDelegate`.
- Removed the `AppLaunchCoordinator` layer, showing up that some times, more layers is not better code. More if these layers are simple forwards from one site to other.
- Fix the bad singleton creation making our `AppDelegate` even simpler.

We should question always to ourselves when we're going to add a new layer or new component, if it give us something. Not always have more layers it's meaning that our code it's more abstrated or decoupled. Each layer we add to our code it's adding complexity and more code to maintain and that can make our code hard to refactor in the future.