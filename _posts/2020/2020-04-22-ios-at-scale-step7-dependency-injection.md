---
layout: post
title: Xcode at scale - Dependency Injection - flexible and composable
author: juli
---

This is the eighth step of a serie named **iOS at Scale** based on the next steps:
- [Introduction]({% link _posts/2020/2020-04-15-ios-at-scale-step0-introduction.md %})
- [Xcode at Scale]({% link _posts/2020/2020-04-16-ios-at-scale-step1-remove-xcode-conflicts.md %})
- [Refactor MarvelClient - split client from logic]({% link _posts/2020/2020-04-17-ios-at-scale-step2-refactor-marvel-client.md %})
- [Refactor DataProvidersKit - applying Iversion of Control]({% link _posts/2020/2020-04-18-ios-at-scale-step3-refactor-data-providers-kit.md %})
- [Refactor DetailKit - Single data flow, states, type erasure and more]({% link _posts/2020/2020-04-19-ios-at-scale-step4-single-data-flow.md %})
- [Refactor Navigator - Back to simplest]({% link _posts/2020/2020-04-20-ios-at-scale-step5-refactor-navigator-kit.md %})
- [Refactor AppCore - we really need it?]({% link _posts/2020/2020-04-21-ios-at-scale-step6-app-core-module.md %})
- [Dependency Injection - flexible and composable]({% link _posts/2020/2020-04-22-ios-at-scale-step7-dependency-injection.md %})
- [Extra ball: Cache made easy]({% link _posts/2020/2020-04-23-ios-at-scale-step8-cache-made-easy.md %})
- [Extra ball: Introducing to Promises 1]({% link _posts/2020/2020-04-24-ios-at-scale-step9-introduce-promises.md %})
- [Extra ball: Introducing to Promises 2]({% link _posts/2020/2020-04-25-ios-at-scale-step10-refactor-promises.md %})
- [Final conclusions]({% link _posts/2020/2020-04-26-ios-at-scale-step11-conclusions.md %})


From the very beginning, Dependency Injection is one of the things that I really enjoy doing when developping. From my point of view, is one of the basis of programming, the basis on which are built all of the good software practices. DI it's here to facilitate us replace components by test doubles, makes the principle of Interface Segregation take importance, because is by DI how we can pass objects according to an interface without pain to substitute for others. DI not only will help with Liskov Principle, but also will help to reuse little components along all the code base. I can't imagine daily programming without DI.

To do so, we have available a lot of patterns. We can inject by constructor, by properties, by methods, using anotations, `Factories` [ServiceLocator](https://en.wikipedia.org/wiki/Service_locator_pattern)... Since Swift have default parameters, it's very easy to do DI with those default parameters, and at least have the option to pass other instance if necesary. We can even DI based on default parameters plus Singleton Instances. On the other hand there are libraries or frameworks like [Needle](https://github.com/uber/needle) that generates code for us. Other libraries are container based where we provide some registrations to after retrieve components. Those registrations are little factories that combining them we build bigger components. But _yo he venido aquí a hablar de mi libro_ (sorry, spanish joke, the translation is I've come here to talk about my book).

Maybe like I started my developer life working on `Java`, with `Spring`, I'm used to get help with a tool with this task. I really like how `Components`, `Services` and `Autowire` works togheter on that platform, I also remember how we must define the graph on an _xml_ file based on class names `Strings`, and how this could stop working only with a typo (that it's pain).

In this post I'm going to intruce [Injection](https://github.com/julianalonso/injection), a lightwight library for dependency injection. [Injection](https://github.com/julianalonso/injection) it's built with the idea of be able to reuse your code pieces in an easy way, build them with factories, group that factories on `Components` (if you want), and provides an standard component to build your Screens named `ModuleBuilders`.

The way you work with [Injection](https://github.com/julianalonso/injection) it's providing on the application launch the `Components`/`Factories` that you want to share, and then, creating your `ModuleBuilder`s to build your `Screens`. With these ModuleBuliders on the game, we gain the option to pass to our instances run time parameters, **named** and **typed**.

This library, fits very well with the [Navigator](https://jobandtalent.engineering/the-navigator-420b24fc57da) component, since the `Screen` creation it's only a function that returns a `ViewController` and our `ModuleBuilders` are created with the purpose of return the Root thing you have on your application.

[Injection](https://github.com/julianalonso/injection) its on early stage of development, but meets the basics to be working on production apps. And it counts with a DSL very easy to read. Mainly insipred on [Koin](https://insert-koin.io/).

Like we've seen on previous post, on of the main sources on pain of the code base, has been the DI. Not because an Assembly object will be always a pain, not. It has been a source of pain because it has a lot of variables and interfaces propagated along all the modules that makes to me the dependency graph hard to follow when refactoring. Also these assemblies has mixed dependencies from several modules, and they was stored on memory after creation, instead of beign built, create whatever thing they must create and die.

So let's update the project to use [Injection](https://github.com/julianalonso/injection).

## Refactor how Dependency Injection it's done

Move to step-6 tag on git.

The first thing we need to do it's add [Injection](https://github.com/julianalonso/injection) as dependency to the project with [SPM](https://swift.org/package-manager/). Like [SPM](https://swift.org/package-manager/) builts static libraries, we only can add this dependency to one framework of the app. Like I want to use the Features Modules Screens, in an easy way from other features, and the definition of others app components I want do it on the app, I'll add the depedency to DisplayKit. This way I have it available on the features among the App. (commit: `6813496`)

Now with injection added, I'm going to start creating our `Injection` `Components` by layer. One for each layer of the core components. The first component to do is the `MarvelClient` component. We can see on the `Assembly` of this component, how we have writed the keys to connect with the Marvel API, with a comment _`//This should be in other part`_, and yes, it should be.

I've created an `Environment` file that it's located on `App/Sources/Environment.swift`, this file could be ignored on git, create an `App/Sources/Environment.swift.template` for example, and with [Makefile](https://opensource.com/article/18/8/what-how-makefile) or other automatizing tool like [fastlane](https://fastlane.tools/) or a simple bash script, create a command to, if there's no `Environment.swift` provided, copy the template, and then generate the project. This way, our developers only has to run a command on the shell to have the project working and we're not sharing our secrets. (You may find an implementation of this command [here](https://github.com/JulianAlonso/MarvelApp/blob/master/scripts/setup.sh))

Finally, our MarvelClient component will look like: (commit: `32fc02f`)

```swift
let marvelComponent = Component {
    factory { Authorization(publicKey: Environment.publicKey, privateKey: Environment.privateKey) as Authorizating }
    single { HTTPClient(host: Environment.host, session: .shared, authorization: $0()) as HTTPPerforming }
    factory { CharacterService(client: $0()) as CharacterServicing }
    factory { CharacterProvider(service: $0()) as CharacterProviding }
}
```

As we may see, the dependencies of `CharacterService` or `CharacterProvider` are given by the `$0()`. This it's the `resove()` function on the module where the `Component` is registered, this way, you can have the same component but change the dependencies depending of the `Module` where they are registered. You will see in action later.

Now that we have our client component done, let's do the `Repository` layer `Component`. (commit: `bd31fe0`)
This repository as you can see is very small, and we could remove it to use a simple `factory`, but if the application grows, the most possible thing it's to have more than one repository.

```swift
let repositoryComponent = Component {
    factory { InternalCharacterRepository(provider: $0()) as CharacterRepository }
}
```

Now, with those components created, we're going to provide them to injection to make them available inside the `ModuleBuilders`. This must be done whenever you want before instanciate your `Screens` by `ModuleBuilders`, so I suggest you to do it on the `AppDelegate`, and it will look like: (commit: `34f0d9d`)

```swift
injectMe {
    component { marvelComponent }
    component { repositoryComponent }
}
```

That creates a shared `Module` and provides it to [Injection](https://github.com/JulianAlonso/Injection). This module will be taken on the `ModuleBuilder`s where you also will have the option to extend it with another `Component`.

Mow we have available those components inside `ModuleBuilders`, so let's create our first `ModuleBuilder` for the Loading and Retry ViewController. (commit: `9bd9238`)

```swift
public final class LoadingModuleBuilder: ModuleBuilder<UIViewController> {
    
    private let message: String
    
    public init(message: String) {
        self.message = message
    }
    
    public override func build() -> UIViewController {
        LoadingViewController(detailText: message)
    }
}
```

```swift
public final class RetryModuleBuilder: ModuleBuilder<UIViewController> {
    private let title: String
    private let description: String
    
    public init(title: String, description: String) {
        self.title = title
        self.description = description
    }
    
    public override func build() -> UIViewController {
        RetryViewController(title: title, descriptionText: description)
    }
}
```

As we see, `RetryViewController` has a delegate with only one function, in my opinion, we can simplify delegation by functions, like in this case, we can pass it by the constructor, like we'll see. (commit: `109e553`)

Then, let's create the DetailContainer module builder and the detail module builder. Once created, we could remove the assembly. Then, update the dependency on the app core assembly, because we're not using Assembly anymore on character detail screen. The module builder it's reponsible of build all dependencies for one `Screen`, this way, we have localized where those dependencies are created or provided, but keeping the flexibility of use dependencies provided by `injectMe`.

Also, if we want to reuse some view or some view model for other `Screen`, we can create another `ModuleBuilder`, and reuse those compoenents. Each module builder it's reponsible to build one screen, also we have the option to override `injectMe` components if needed simply adding to the Component the new factory for the types we want to override. This way, we have the granularity needed to make each screen as we need, while avoiding to have screen components assemblies instanciated on memory like we have before.

Now we also simplified how the screen creation it's done, because we've removed the assembly-provider needs.
(commit: `cbd0271`)

```swift
public extension Screen {
    static func detail(character id: CharacterId) -> Screen {
        .init { CharacterDetailContainerModuleBuilder(characterId: id).build() }
    }
}
```

Now that we have removed all the detail assemblies, we can remove the providers interfaces, and the character detail view controller factory. Also we've removed the formatter dependency on the Detail that was not used. (commit: `000c156`)

We keep having the ability of update how Loading or Retry ViewControllers are built by updating the module builder, but this time the creation of these view controllers, can be done on the container directly instead having a factory. Our ModuleBuilders are factories by themselves.

Also, our [Injection](https://github.com/JulianAlonso/Injection) code keeps out of our code base, making Injection easy reemplazable with other library, or our custom DI, or simply to don't make all our instances highly depends on what features Injection provides.
Before, we have the Assemblies on the modules, and features was depending on those modules very hard, now it's our app who choices about what instance to use for what dependency. And testing we can inherit from the app module, and override as many factories we need for the test. (We can for example, provide a fake `HTTPPerforming`, that instead going to network, give us stubs for requests)

Note: _like the app assembly it's a Singleton, with the others Assemblies like lazy vars, if our app grows, we're loading on memory a lot of assemblies that won't be released, this will cause our app run with a lot of memory while the user navigates though our app, so that implementation won't scale so fine_

Now let's do the same for the List Module. As we see, we've reduced a lot the complexity of the `Assemblies`. (commit: `3db140d`).

At this point, we've lost the `Navgation` between the list and the detail. This is cause about how we're doing the navigation, we will fix this after, now let's update the app core assembly.

To update the `Assembly` on the app core, before I'll refactor how the `navigation` components are named, only `Navigator`, to `Navigating` to express that it's a protocol. Then remove the old `NavigatorKitAssembly` and the `AppCoreKitAssembly`.

And now, we got accesible on all `ModuleBuilders` on our app the `Navigator`, and due it's a `single` it will be always the same instance. (`single` are the factories for `Singleton` instances)

```swift
final class AppDelegate: UIResponder, UIApplicationDelegate {
    
    var window: UIWindow?
    
    public func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        self.window = UIWindow(frame: UIScreen.main.bounds)
        let navigator: Navigating = Navigator(window: window!)
        
        injectMe {
            component { marvelComponent }
            component { repositoryComponent }
            component { uiComponent }
            single { navigator }
        }
        
        navigator.handle(navigation: .root(.list))
        return true
    }
    
}
```

Now that we have the navigation made simpler. Let's do navigation between features easy and painless.
The only target that has all the `features` available it's the app target, so in that target, we will need to handle the `navigation`. And we will do this with protocols, to be easy injectables, and replaceable for test doubles if needed.

And then, we will be using the component modularization provided by Injection to keep this clean.

We need a protocol for the list navigation, and the component proposed to this will be a `Coordinator`.

Let's create the `CharacterListCoordinator`: (commit: `a9a6d77`)

```swift
public protocol CharacterListCoordinating {
    func characterDetail(id: Int)
}
```

And then, we create a base `Coordinator` to share the simple initializer with the `Navigator` component.

```swift
class Coordinator {
    
    let navigator: Navigating
    
    init(navigator: Navigating) {
        self.navigator = navigator
    }
    
}
```

Then, the character list coordinator.
```swift
final class CharacterListCoordinator: Coordinator, CharacterListCoordinating {
    func characterDetail(id: Int) {
        navigator.handle(navigation: .push(.detail(character: id)))
    }
}
```

And the last, we're going to create a `Injection` `coordinatorsComponent` to provide this instances to the features. (commit: `1b3cfe7`)

```swift
let coordinatorsComponent = Component {
    factory { CharacterListCoordinator(navigator: $0()) as CharacterListCoordinating }
}
```

This components doesn't need to be a singleton instance, since them use the navigator provided by the DI, and really, they only wire features, the don't know anything about the `Navigation`. 
Also keep beeing simple pieces of code that only wire two features. These pieces are simples and easy to maintain, but keep in mind that in a big application the will be a lot.

Now we have to update the ModuleBuilder of the presenter that performs the navigation. (commit: `3acc3ad`)

```swift
// CharacterListPresenter
func characterSelected(at index: Int) {
    let character = characters[index]
    coordinator.characterDetail(id: character.id)
}
```

Inclusive now, we can even remove the `NavigationKit` dependency from the `Features` if we want, and move the extensions of the `Screen` to the app target if we want.

This way we have our features even more decoupled of the `NavigatorKit` component. But I like to see the `Screen` definition under the `ModuleBuilder`, because we're going to have a 1:1 relation from `ModuleBuilder` to `Screen`. So if we find one `ModuleBuilder` without Screen definition, maybe it's a signal that we lost something.

## Conclusions

In this chaper we've reduced the complexity subyacent to the `Assemblies`. Not because having `Factories` with `ServiceLocator` it's a bad pattern. Because them was based on many protocols, without structure inside them and was making very hard to read the depedency graph.

We are clearer no about how the navigation it's done with `Coordinators` that are easy pieces of code and allow us to customize navigation as many as we want.

Now, we have a very simple navigation beetween screens, all coordinators are built in the same way, and all navigations are performed by the `Navigator` but inside the app target, the responsible to built this navigation.

Now we saw that we can change the `Navigator` for other component if we want in a very easy way, and we have all our pieces of code easy to read, to maintain and to perform further refactors like the two next chapters.

I introduced `Injection`, but you can use other library, on simply use Service Locator / Factory pattens. Our code base won't be better for use one tool or other, it will depends on what we're doing.

If you want to give a try to `Injection`, I encorage you to it. You can group your factories by modules also, define those modules where you want, combine and share them. With `MdouleBuilders` you'll have factories to build your screens with needed dependencies, and you can create as many Modules as you need. Since Components are not other thing than an array of functions, they will not use to much memory beign variables out of scope as we already saw here. Modules are where factories functions are mutated to objects if needed, this will happen only on `single` (until new type of factories arrive), the instance factory `factory` it's an struct with the function to build the instance, so really it won't use so much memory neither.

## Steps

- [Introduction]({% link _posts/2020/2020-04-15-ios-at-scale-step0-introduction.md %})
- [Xcode at Scale]({% link _posts/2020/2020-04-16-ios-at-scale-step1-remove-xcode-conflicts.md %})
- [Refactor MarvelClient - split client from logic]({% link _posts/2020/2020-04-17-ios-at-scale-step2-refactor-marvel-client.md %})
- [Refactor DataProvidersKit - applying Iversion of Control]({% link _posts/2020/2020-04-18-ios-at-scale-step3-refactor-data-providers-kit.md %})
- [Refactor DetailKit - Single data flow, states, type erasure and more]({% link _posts/2020/2020-04-19-ios-at-scale-step4-single-data-flow.md %})
- [Refactor Navigator - Back to simplest]({% link _posts/2020/2020-04-20-ios-at-scale-step5-refactor-navigator-kit.md %})
- [Refactor AppCore - we really need it?]({% link _posts/2020/2020-04-21-ios-at-scale-step6-app-core-module.md %})
- [Dependency Injection - flexible and composable]({% link _posts/2020/2020-04-22-ios-at-scale-step7-dependency-injection.md %})
- [Extra ball: Cache made easy]({% link _posts/2020/2020-04-23-ios-at-scale-step8-cache-made-easy.md %})
- [Extra ball: Introducing to Promises 1]({% link _posts/2020/2020-04-24-ios-at-scale-step9-introduce-promises.md %})
- [Extra ball: Introducing to Promises 2]({% link _posts/2020/2020-04-25-ios-at-scale-step10-refactor-promises.md %})
- [Final conclusions]({% link _posts/2020/2020-04-26-ios-at-scale-step11-conclusions.md %})