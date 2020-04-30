---
layout: post
title: Xcode at scale - Refactor Navigator - Back to simplest
author: juli
---

This is the sixth step of a serie named **iOS at Scale** based on the next steps:
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


The Navigator component proposed by [Job & Talent ingeenering team]() it's a piece I've falled in love with it sice I've read the post.

This component handles the navigation of the app easy. It simplifies how our app navigates in a very easy way, having a bunch of options of navigation expressed by an `enum`, that allow us to use the dot notation. (`.push(.screen)`) (nontation that I love).

I guess the simplicity of the component it's the main key feature. It allow us to perform all the possibles navigations while keeps the navigation centralized on one single component, and we're abstracted about how it's performed and the stack. This component could be extended as many as we can, since we only need to make more navigation options.

I made a PoC about the `Navigator` a couple of days after read about it, and I introduced it to [Devengo](https://www.devengo.com/) app, as soon as I can, because despite being simple, it's really powerful of how it does the navigation, and I love how it could be combined with [Injection](https://github.com/JulianAlonso/Injection) but we will talk about this on the next post.

In this post, I'm only going to remove the layer of the `NavigationDelegate` inside the features, by moving the dependency of the `Navigator` to the DisplayKit module. Like the Navigator it's a piece of code already abstracted, that allow us to keep refactoring on the future how it performs navigation.

Also we'll findout that our features frameworks already depends on the `NavigatorKit`, but the dependency it's not declared on them. This could lead us to errors, since we're not beign explicit of what dependency it's using what module. This is something that we should highly avoid.

## Moving the dependency

Since we're using [Tuist](tuist.io), this step it's really easy, because we only need to get out the dependency from the `AppCore` `Manifest` and move it to `DisplayKit`. (commit: `3f1b27f`)

We can see how doing this, hasn't got any impact on the app. It keeps compiling at the same time that keep's the navigation working. But we've gained the option to extend the `Screen`.

Now, time to refactor a little bit the navigator.

## Refactor the Navigator

As we can see, this implementation has the `Screen` as protocol it's making us lose some key points of the `Navigator`. The `Screen` as struct, like the article says, it's to have the ability of extend it with our screens providing `static functions` that build's up our `Screen` objects. Having static funtions, as any other function, will have the ability of receive parameters, and those parameteres will be the needs of the screen to be built, **named** and **typed**.

Since we're not working on JavaScript, I highly suggest you to avoid pass parameters by dictionaries that we could pass by functions. Passing parameters by dictionaries could lead us to forget what parameteres are needed, what type they should be and make our application crash on runtime, harder to maintain, harder to refactor and harder to understand.

And not forget that with static functions that returns the same type where they are implemented (eg: extend's UIColor with static colors), we gain the dot notation, other key point of the `Navigation` simplicity, plus autocomplete option.

So let's refactor the `Screen` to: (commit: `60050a1`)

```swift
public struct Screen {
    public let build: () -> UIViewController
    
    public init(_ builder: @escaping () -> UIViewController) {
        self.build = builder
    }
}
```

And now, let's update how the `Navigator` and `Navigation` works according to this new Screen type.

```swift
public enum Navigation {
    case root(Screen)
    case push(Screen)
    case present(Screen)
}

public protocol Navigator {
    func handle(navigation: Navigation, animated: Bool)
}

public extension Navigator {
    func handle(navigation: Navigation) {
        handle(navigation: navigation, animated: true)
    }
}

final class InternalNavigator: Navigator {
    private let window: UIWindow
    private var navigationController: UINavigationController!
    
    init(window: UIWindow) {
        self.window = window
    }
    
    func handle(navigation: Navigation, animated: Bool = true) {
        switch navigation {
        case let .root(screen):
            setRootScreen(screen)
        case let .present(screen):
            presentationViewController().present(
                screen.build(),
                animated: animated
            )
        case let .push(screen):
            navigationController.pushViewController(
                screen.build(),
                animated: animated
            )
        }
    }
}

private extension InternalNavigator {
    func setRootScreen(_ screen: Screen) {
        navigationController = UINavigationController(rootViewController: screen.build())
        window.rootViewController = navigationController
        window.makeKeyAndVisible()
    }
    
    func presentationViewController() -> UIViewController{
        if let vc = navigationController.visibleViewController {
            return vc
        }
        
        if let vc = navigationController.topViewController {
            return vc
        }
        
        return navigationController.viewControllers.last!
    }
}
```

Now, let's simplify a little bit more the `Navigator`. I really like has as variables things that they could be variables, so the presentationViewController could be a vaiable. But this is only personal preferences.

And now, our presentationViewController will look like: (commit: `7a1b6c3`)

```swift
var presentationViewController: UIViewController {
    guard let vc = navigationController.visibleViewController ??
        navigationController.topViewController ??
        navigationController.viewControllers.last
        else {
        fatalError("Navigator with empty NavigationController")
    }
    return vc
}
```

Now, we need to update the provided screens to make them work with the new `Screen` struct.
So let's transform this code:

```swift
/// CharacterDetailScreen in the main screen for the character detail feature
class CharacterDetailScreen: Screen {
    enum Params {
        static let characterId = "CharacterId"
    }
    
    private unowned let characterDetailContainerViewControllerProvider: CharacterDetailContainerViewControllerProvider
    
    init(characterDetailContainerViewControllerProvider: CharacterDetailContainerViewControllerProvider) {
        self.characterDetailContainerViewControllerProvider = characterDetailContainerViewControllerProvider
    }
    
    func viewController(with params: ScreenParams?) -> UIViewController {
        guard let characterId = params?[Params.characterId] as? CharacterId else {
            fatalError("Cant navigate to details without an ID")
        }
        return characterDetailContainerViewControllerProvider.characterDetailContainerViewController(characterId: characterId)
    }
}
```

To this other code: (commit: `63f02de`)

```swift
extension Screen {
    static func detail(provider: CharacterDetailContainerViewControllerProvider,
                       character id: CharacterId) -> Screen {
        .init { provider.characterDetailContainerViewController(characterId: id) }
    }
}
```

Note: _As we've seen, having parameters inside dictionaries, are a really bad option._

And now, let's remove the `CharacterDetailNavigator` (commit: `fbf7e79`). At this moment, we can see the highly coupling between modules that we have. We've removed an interface from `DisplayKit` that was pointing to `DetailKit` (this means that `DisplayKit` has an interface for each screen of our app, if we have a big app, we have a lot of clases inside `DisplayKit` that we shouldn't have because `Displakit` isn't dependening on our features). 

Now, our `CharacterListKit` also stop compiling, since it also need to be adapted to the new `Screen` type. The first it's to update his `Screen`, and the also remove the custom navigator.

We pass from this:

```swift
class CharactersListScreen: Screen {
    private unowned let charactersListContainerViewControllerProvider: CharactersListContainerViewControllerProvider
    
    init(charactersListContainerViewControllerProvider: CharactersListContainerViewControllerProvider) {
        self.charactersListContainerViewControllerProvider = charactersListContainerViewControllerProvider
    }
    
    func viewController(with params: ScreenParams?) -> UIViewController {
        return charactersListContainerViewControllerProvider.charactersListContainerViewController()
    }
}
```

To: (commit: `3fc3dde`)

```swift
extension Screen {
    static func list(provider: CharactersListContainerViewControllerProvider) -> Screen {
        .init { provider.charactersListContainerViewController() }
    }
}
```

Now let's fix the assembly. Note that we've removed the `NavigatorProvider`, because we're using already a function that give us the screen, that simply functions do the same thing that the `NavigationProvider` was doing, create the `Screen` object. So again, we reduced an extra complexity layer.

Note also that this assembly, that it's here to create the List `Screen` with his dependencies, has the context of knowing that this screen is the main screen of the app. Something that we should avoid, because this could change over the app scales. So I'm going to remove the next piece of code.

```swift
public var mainScreen : Screen {
    CharactersListScreen(charactersListContainerViewControllerProvider: self)
}
```

Following with this, the `Assembly` of the `ListPresenter` has the dependency of the `DetailNavigator`, so let's update how the navigation it's done by simply doing a function that will performs the navigation (this is a bad option too, but due to the coupling of the layers, it's hard to me follow all the code. This is fixed on the Dependency Injection post, so please, be patient). (commit: `6d968eb`)

Now we need to fix the `AppCoreKitAssembly` according to this new navigation and `Screen` construction. Now we will se, that instead of passing the `CharacterDetailContainerViewControllerProvider` we may pass the `Assembly` that it's the thing that provide us the `ViewController`, and avoid us to make a lot of components public, because it it's already public.

Our detail screen becomes to be:

```swift
public extension Screen {
    static func detail(assembly: CharacterDetailKitAssembly,
                       character id: CharacterId) -> Screen {
        .init { assembly.characterDetailContainerViewController(characterId: id) }
    }
}
```

After fix the `AppCoreAssembly`, we will find us on commit: `687ed3d`. Now, we're capturing our `AppCoreKit` that could make you think that now it stills on memory, but remember, that assembly it's a singleton, so it keeps already on memory. We will fix the Assemblies in memory problems on the Dependency Injection chapter.

Now we need to tell the app launch coordinator, what it's the first screen, but now, we're defining on the `AppCoreKit` what Screen is the first screen, while our `AppListAssembly` keeps without know if it is the first screen or not.

Again, we need to make our `Screen` extension public. And instead of passing the provider, lets give it the assembly directly. Now, our app it's compiling again (commit: `f1b8284`)

And finally, we can remove the two `ViewControllerProvider` protocols, since we're not using them on our screens functions. (commit: `e5b64ae`).

## Conclusions

Let's think about what we've done here. 

We've maked easier to read and easier to extend the `Screen` component of the `Navigator` and also removed the custom `Navigator` layer on the listkit to a function that navigates (again, this is not the best option, we will see how fix this on Dependency Injection chapter).

We've removed the dictionary of properties that we should pass on the `Screen` instance, that was hidding what properties the screen needs and the type of those properties, and this should avoid this as soon as we can. Note that having that implementation on a big app, could lead us to inumerable errors.

One of the most hard steps inside each refactor we're doing, it's how the dependency injection it's done, it has a provider protocol for each component that it's beign built on the app, making the assembly a piece of code very hard to read and modify, as you can see on the posts, I'm fixing the assembly based on the bugs that gives me the compiler but without fully understand what it's building the assembly. Have a provider protocol for each component, it's adding complexity but like we see, doesn't give us the flexibility of perform refactors with painless, because even with interfaces, the components are very very coupled and makes hard to follow the dependency graph.





