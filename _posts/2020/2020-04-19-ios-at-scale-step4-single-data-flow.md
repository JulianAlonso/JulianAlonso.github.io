---
layout: post
title: Xcode at scale - Refactor Detail kit - Single data flow, states, type erasure and more
author: juli
---

This is the fifth step of a serie named **iOS at Scale** based on the next steps:
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


Finally, time for the Features modules. Go to the `step-3` on the Project.

This modules are designed with a very custom implementation of VIPER. VIPER itself it's an implementation of Clean Architecture proposed by [objc.io](https://www.objc.io/issues/13-architecture/viper/) on the old (_not so good_) objc times. This architecture lead us to have a big explosion of classes and interfaces, that in objc without generics and today tools, could be a good option to do scalable and mainteinable applications, but nowadays it introduces a lot of complexity that could be removed with new techniques like we'll see in this post.

I'm not saying that VIPER it's a bad architecture. VIPER lead us to have every layer of the Views abstracted, our views to the presenter are wired by interfaces, the presenter to the view also wired by interface. This make our applications easy to test, replacing our componentns by test doubles it's really easy if we follow Interface Segregation, like VIPER follows. But, how many times we start creating a module of our app, and we lose easily about 15 minutes only writing boilerplate code?

Boilerplate code it's very usual to have on every applicacion with a good design, and there're tools that handle this for us, creating Modules based on a template. I remember me using [Viper-module-generator](https://github.com/redbooth/viper-module-generator), feature that it's today included on [Tuist](https://tuist.io/docs/commands/scaffold/) with the command `tuist scaffold`.

But what if I told you that we can archieve the same abstraction between the `View` and the `Presenter` with less boilerplate?.

Thank's to the state based views, we will se how we can create a single data flow application, with [Protocol Oriented Programming](https://developer.apple.com/videos/play/wwdc2015/408/), reduces our complexity, making the view-presenter communitcation simpler, but at the same time easy reemplazable by test doubles.

Also, as we can see on the given project, we will found `States`. Make our views responding to `State`s will make our life easier, since we start to modeling possible states, and our inconsistent `States` that could not be produced, are imposibble to create. 

Note: _that inconsisten states are when we have two optionals that we can't really have both, it's one or other. This cases nowadays could be modeled with enums and associated values_

We have enum expressing these possible States, with enums we win for example, there's no state loading and with data, we end up with a bunch of options that represents every state of a single View and also limit us to pass to the view impossible states, sice we can only give one of the enum options. But, we will see like `States` could not be only enums, for views with a single state, we can use structs, since those views doesn't has more than one possible state.

In this step, I'm going to refactor only the `CharacterDetailKit`. Thanks to Feature Freamework architecture, we can have each module of our app with different architectures and still have our app working. one of the high benefits of this it that we could make incremental refactors on our app like we are seeing on this post series.

## What we're going to do?

Lets simplify our View and Presenter comunications with one `Protocol` and keep our View's responding to an state, this way, we only can update the views with a [single data flow](https://www.geeksforgeeks.org/unidirectional-data-flow/), and only one ouput method, where we going to pass an enum. This output method will be an `Action` with parameters of the action as associated values (if needed).

This new kind of communication often it's more related to a `ViewModel` than a `Presenter`, and to try keep away from Viper, I'm going to  create our `ViewModel`. This `ViewModel` will be on `DisplayKit`, because I wan't to make it available along all the Features. But if you don't need to, you can have it on your feature module. 

So that's given, we need a `ViewModel` that handle `Action`, and a `View` that's reponds to an `State`.

```swift
public protocol StatefulView: AnyObject {
    associatedtype State
    
    func render(state: State)
}

open class ViewModel<State, Action> {
    
    public private(set) var state: State
    
    public init(state: State) {
        self.state = state
    }
    
    open func handle(_ action: Action) { fatalError("Calling not implemented handle") }
    
    final public func update(state: State) {
        self.state = state
    }
    
}
```

As we see, the `View` protocol has associated types, so we can't not use them in our View/ViewModels implementations to have references to it. This time we need to use [Type Erasure](https://medium.com/@chris_dus/type-erasure-in-swift-84480c807534). If you're not familiar with it I highly suggest you to read and try to understand how it works because it's really powerful.

Type erasure lead us to create an object with the prefix `Any` and the protocol we want to avoid `associatedtype`, in this case `StatefulView`. Then conform the protocol, and set the associated type of the protocol as `Generic` on our `Any` object. This way, we can abstract about what object it's conforming the protocol. We're wrapping the `StatefulView` object on `AnyStatefulView`, to have the option of use it as generic constraint, something that we can't do with a protocol with associated values. 

With this, we're expressing on our `ViewModels` that we don't care about what object it's conforming the protocol. We declare, ey! I need something confonrming `StatefulView` wich this given `T` `State`.

```swift
public final class AnyStatefulView<State>: StatefulView {
    
    private let _render: (State) -> Void
    
    public init<V: StatefulView>(view: V) where V.State == State {
        _render = { [weak view] state in view.render(state: state) }
    }
    
    public func render(state: State) {
        DispatchQueue.main.async { self._render(state) }
    }
    
}
```

Now we got an "abstract" type to reference our views from our `ViewModels`, as you can see now the `ViewModel` it's a base class, but we don't want our `View` referencing to a concrete type of our `ViewModel`, so we're going to apply `Type Erasure` again but this time to a class. I know Type Erause it's designed to work with protocols, but this time will also help us. (Really it's a simple wrapper, but that it's what really Type Erasure is). 

```swift
public final class AnyViewModel<State, Action> {
    
    var state: State {
        get { viewModel.state }
    }
    private let viewModel: ViewModel<State, Action>
    
    public init(viewModel: ViewModel<State, Action>) {
        self.viewModel = viewModel
    }
    
    public func handle(_ action: Action) {
        viewModel.handle(action)
    }
    
}
```

Now we got our future views decoupled from our `ViewModel` implementation. But our `ViewModel` are not linked yet to the views.
The way we're going to link them its by subscription. We are going to pass a view to the `ViewModel` and the `ViewModel` will call our `View.render` method. So let's add the subscription method.

```swift
open class ViewModel<State, Action> {
    
    public private(set) var state: State
    private var views = Set<AnyStatefulView<State>>()
    
    public init(state: State) {
        self.state = state
    }
    
    open func handle(_ action: Action) { fatalError("Calling not implemented handle") }
    
    final public func update(state: State) {
        self.state = state
        views.forEach { $0.render(state: state) }
    }
    
    final public func subscribe<V: StatefulView>(view: V) where V.State == State {
        views.insert(AnyStatefulView(view: view))
    }
}
```

Now, to insert `AnyView` inside a Set, we need to make them `Hashable`. (commit: `e07323b`)

```swift
çpublic final class AnyStatefulView<State>: StatefulView {
    
    private let _render: (State) -> Void
    private let identifier: String
    
    public init<V: StatefulView>(view: V) where V.State == State {
        _render = { [weak view] state in view.render(state: $0) }
        identifier = String(describing: view)
    }
    
    public func render(state: State) {
        DispatchQueue.main.async { self._render(state) }
    }
    
}

extension AnyView: Hashable {
    public func hash(into hasher: inout Hasher) {
        hasher.combine(identifier)
    }
    
    public static func == (lhs: AnyView, rhs: AnyView) -> Bool {
        lhs.identifier == rhs.identifier
    }
}
```

And now, the only thing we need to do, it's to make avaliable on the `AnyViewModel` the subscription method to the `ViewModel`. (commit: `843dc86`)

That we've seen here, it's a very lightweight implementation of [ios J&T architecture](https://github.com/jobandtalent/Lwift), note that this is a very simpler implementation but the concepts of how the data it's going from the `ViewModel` to the `View` and how the `ViewModel` it's responding to `Actions` it's the same, although we don't have `Stores`, render policies and so on.
You can find how that architecture works on [post 1](https://jobandtalent.engineering/ios-architecture-an-state-container-based-approach-4f1a9b00b82e), [post 2](https://jobandtalent.engineering/ios-architecture-separating-logic-from-effects-7629cb763352).

With our `ViewModel` and `View` ready, lets start making the communication with the `CharacterDetailViewController` and it's `Presenter`
Since now our DetailViewController will responds to an `state`, we can remove the interface `CharacterDetailView`. Then make our `ViewController` conform `StatefulView` rendering the state `CharacterDetailState`. (commit: `72fc46d`)

```swift
protocol CharacterDetailContainerView: AnyObject {
    func showView(forState state: CharacterDetailState)
}
```

Now it's time to refactor our `Presenter` to `ViewModel`. Like we'll need not only the `StateType`, but also the `Action` type, let's also create an enum to this Action, named `CharacterDetailAction` with `.load` member. (commit: `fd7c44d`)

```swift
enum CharacterDetailAction {
    case load
}
```

The final look of our ViewModel will be: (commit: `737d09f`)

```swift
final class CharacterDetailViewModel: ViewModel<CharacterDetailState, CharacterDetailAction> {
    private let getCharacterDetail: GetCharacterDetail
    private let characterId: CharacterId
    
    private var searchString: String?
    private var offset = 0
    
    init(state: CharacterDetailState,
         getCharacterDetail: GetCharacterDetail,
         characterId: CharacterId) {
        self.getCharacterDetail = getCharacterDetail
        self.characterId = characterId
        super.init(state: state)
    }
    
    override func handle(_ action: CharacterDetailAction) {
        switch action {
        case .load: load()
        }
    }
}

private extension CharacterDetailViewModel {
    func loadCharacterDetail() {
        getCharacterDetail.execute(with: characterId) { result in
            switch result {
            case .success(let characterDetail):
                self.getCharactersFinished(with: characterDetail)
            case .failure(let error):
                self.getCharactersFinished(with: error)
            }
        }
    }
    
    func getCharactersFinished(with characterDetail: CharacterDetail) {
        update(state: .loaded(characterDetail))
    }
    
    func getCharactersFinished(with error: CharacterDetailError) {
        update(state: .loadError(title: "Something went wrong", description: error.localizedDescription, delegate: self))
    }
    
    func load() {
        update(state: .loading("Loading character detail"))
        loadCharacterDetail()
    }
}

extension CharacterDetailViewModel: RetryViewControllerDelegate {
    func retryViewDidTapOnButton() {
        load()
    }
}
```

Now comming back to the `CharacterDetailVC` let's do the final steps to have it working. We need to update the dependencies of the Presenter to the New `ViewModel`, and inside the create function, we subscribe our ViewController to the `ViewModel`. (commit: `4748380`)
And finally, let's resolve the dependencies broken on the `Assembly`. (commit: `4720f53`)

Since our Container view it's responding to the `State`, let's make also our + responds to a +. This new state will be a `DisplayModel` object. (commit: `8976177`)

```swift
struct CharacterDetailDisplayModel {
    let name: String
    let bio: String
    let image: URL?
}
```

And then, conform StatefulView with that display model as State and create his ViewModel. Like this view will not output actions the action will be `Never`, and our ViewModel will be a simply State holder. We can make our View, render the ViewModel state on the `viewDidLoad` to load the initial state. (Note that we can create a `StatefulViewController` that on his view did load, charge the initial state of the ViewModel, because our ViewModels must be initialized with an initials state, and always have the View with a rendered State). Then fix the dependency graph again and we have it working. (commit: `a521862`)

```swift
final class CharacterDetailViewController: UIViewController {

    @IBOutlet weak var imageView: UIImageView!
    @IBOutlet weak var nameLabel: UILabel!
    @IBOutlet weak var bioLabel: UILabel!
    
    private let viewModel: AnyViewModel<CharacterDetailDisplayModel, Never>
    
    init(viewModel: AnyViewModel<CharacterDetailDisplayModel, Never>) {
        self.viewModel = viewModel
        super.init(nibName: "CharacterDetailViewController", bundle: Bundle(for: CharacterDetailViewController.self))
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        self.render(state: viewModel.state)
        imageView.accessibilityIdentifier = "CharacterDetailViewController"
    }
}

extension CharacterDetailViewController: StatefulView {
    func render(state: CharacterDetailDisplayModel) {
        setUpName(state.name)
        setUpBio(state.bio)
        setUpImage(state.image)
    }
}
```

To end this chapter, the interactor layer. 

Sice our view it's rendering the state on the _main thread_, we may forget about what `Thread` it's our interactor returning things, so we can remove all `Scheduler` related components on our `CharacterDetail` module, fix dependency graph and have our views working. (commit: `6f3b6d6`)

```swift
protocol GetCharacterDetailProtocol: AnyObject {
    func execute(with id: CharacterId, completion: @escaping Done<CharacterDetail, CharacterDetailError>)
}

final class GetCharacterDetail {
    let characterRepository: CharacterRepository
    
    init(characterRepository: CharacterRepository) {
        self.characterRepository = characterRepository
    }
}

extension GetCharacterDetail: GetCharacterDetailProtocol {
    func execute(with id: CharacterId, completion: @escaping Done<CharacterDetail, CharacterDetailError>) {
        characterRepository.character(with: id) { result in
            switch result {
            case .success(let character):
                completion(.success(CharacterDetail(with: character)))
            case .failure(let repoError):
                completion(.failure(CharacterDetailError(characterRepositoryError: repoError)))
            }
        }
    }
}
```

## Conclusions

We swaw how a very easy implementation of `Single Data Flow` with `ViewModels` and _type erasure_, we can archieve reactive views. 

We removed all the complexity of manage subscriptions, that could be a thing a little hard on the begining, but I suggest you to keep learning how them work and take a look to `Combine`, I guess that's the future of how apps will be made. Combine will allow us reduce complexity, write declarative auto explaining code, wire components that will reactive, and many many more things. I've never find a person who time after of working with things like Rx on any platform (if him has understood the tool and how it works), want's to come back to the imperative usual way. And now that Apple it's providing us a framework, it's meaning that its time to study about it.

You may find an [Combine + SwiftUI + reactive ViewModels](https://github.com/JulianAlonso/MarvelApp) example working and showing how we can do the same with so much less code and more declarative.

We made an standard of what method it's called to udpate the `View`, simplifing the view debugging.

We made an `enum` to handle the ouputs of the `View`, and also covered the case where one `View` has no ouput. Note: _we should try to avoid has a `default` case on the handle `switch`, because then when we add a new option to enum's Action, we don't have the compiler error about handle it, and we may find us stuck on errors._

With these changes the data flow it's very easy to follow, and taking a look at the `ViewModel` declaration inside the `View` we will figure out how our view works without effort.

This also help us on our interactors layer, sice our `ViewModels` can do his work 100% asynchronously and only when the `View` it's going to render the state it's when we're back on the _main thread_.

Notice that now, our `ViewModel`s will not have the usual initial inconsistent state because it's mandatory to initialize them with an _state_.

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