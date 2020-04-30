---
layout: post
title: Xcode at scale - Introducing to promises 2 - Refactor to use them
author: juli
---
This is the eleventh step of a serie named **iOS at Scale** based on the next steps:
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


This is our last series post, where we're going to see how promises can make our life a little bit easier.

One of the things that a like about promises it's the function signature with them. We return objects instead having closures as parameters. We return a new kind of object that we can map over. Promises allow us to group them into one, make our code more declarative and read our code easier. 

They can do as many things as we develop on them, and they are generics (one work, a lot of usages). In this case, our promises are very basic, but we're going to see how even a very basic promises help us on our daily basics.

In order to do this, we're going to see another cool feature of [SPM](). At the moment we have two different packages, one for the Promises, and one for the Networking. This time, we're going inside Networking, to create another Target that will be Networking+Promises.

This way, we still have the option to use `Networking` without Promises, and also we can go developing more Promise features if we want. Of course, we have 2 different targets, this means two bundle testing... A lot of benefits.

So let's do it.

## Refactor applying promises

The first thing to do, it's create a new folder hirarchy on the `Networking` folder, like this time we're going to have two targets, we're going to follow the SPM guidelines. We will have `Networking/Networking` and `Networking/NetworkingPromises`. One for the Networking code and the other for the Networking+Promises code.

Then we have to update our `Package.swift`. The Manifest of our package now must indicate the new target, and also we will add the test target for it. (commit: `4dabb6a`)

Now it's time to update our dependency graph inside the project. First, we add the new product as dependency on our dependencies on Tuist. And then we touch the Tuist Manfiest files. Like with other Libraries, we must add it only one time for the project, so we add it to the `Core` (it's our lower package, due that from it depends `MarvelClient`, remember the IoC, and from MarvelClient depends the other packages). (commit: `6d2211c`)

With all set, now we start to work on the `NetworkingPromise.swift` extension. (I did it before, but I show the little work now). 
The only thing we need to do, it's to create a Promise, that perform the request and then unwrap the result to pass the values to the promise.
Like we have done the code of the perform with the old `Done<T, E>`, we can use that and wrap it inside the promise.

```swift
public extension HTTPPerforming {
    func perform<T: Decodable, E: Decodable>(_ endpoint: Endpoint) -> Promise<T, Error<E>> {
        Promise { future in
            self.perform(endpoint) { (result: Result<T, Error<E>>) in
                switch result {
                case .success(let value): future.fulfill(value)
                case .failure(let error): future.reject(error)
                }
            }
        }
    }
}
```

That's it. That's all the code that we need to make our `Networking` library work with promises.

Now, let's start to modifying our services inside the `MarvelClient`. We will `import` now on the service part, not only the `Networking`, but also `NetworkingPromises`and `Promises`. Then change the code of the services to return promises.

We pass from: 

```swift
public func characters(offset: Int?, _ done: @escaping (Result<[Character], MarvelError>) -> Void) {
    client.perform(Endpoint(path: "/v1/public/characters", parameters: ["offset": offset ?? 0])) { (result: Result<Response<[Character]>, Error<MarvelError>>) in
        done(result.map(\.body.results).mapTerror(\.marvelError))
    }
}

public func character(by id: Int, _ done: @escaping (Result<[Character], MarvelError>) -> Void) {
    client.perform(Endpoint(path: "/v1/public/characters/\(id)")) { (result: Result<Response<[Character]>, Error<MarvelError>>) in
        done(result.map(\.body.results).mapTerror(\.marvelError))
    }
}
```


To: (commit: `790091e`)

```swift
public func characters(offset: Int?) -> Promise<[Character], MarvelError> {
    (client.perform(Endpoint(path: "/v1/public/characters", parameters: ["offset": offset ?? 0])) as Promise<Response<[Character]>, Error<MarvelError>>)
    .then { $0.body.results }
    .catch { $0.marvelError }
}

public func character(by id: Int) -> Promise<[Character], MarvelError> {
    (client.perform(Endpoint(path: "/v1/public/characters/\(id)")) as Promise<Response<[Character]>, Error<MarvelError>>)
        .then { $0.body.results }
        .catch { $0.marvelError }
}
```

Now that we have adapted our `Servicing`, let's to adapt the Provider, but this time unwrapping the promise to a `Done` again, to check that the app still working, and show that we may refactor by layers or even method by method while we develop more functionalities, instead refactor the whole app in one time. (commit: `7a24cea`)

```swift
public func characters(offset: Int?, _ done: @escaping Done<[Core.Character], CharacterRepositoryError>) {
    service.characters(offset: offset)
        .then { $0.map(\.coreCharacter) }.then { done(.success($0)) }
        .catch { $0.repositoryError }.catch { done(.failure($0)) }
}

public func character(by id: Int, _ done: @escaping Done<Core.Character, CharacterRepositoryError>) {
    service.character(by: id)
        .then { try $0.first.map(\.coreCharacter) ?? CharacterRepositoryError.notFound }.then { done(.success($0)) }
        .catch { $0.repositoryError }.catch { done(.failure($0)) }
}
```

On that part, we see how we can do steps by `then`. First we take the service promise and map it, and then, we uwrapp the promise and call the done.
This way, we take the promise and come back to the `Done`, and we can do an incremental refactor, if our app it's big. This way of doing refactors of one technologies newer ones it's easy, because we do it layer by layer, or object by object..., and we're nor bound to make a whole app refactor. Or even worse, start an app from zero because our legacy code it's unrefactorizable.

Okey, so now we can continue to end our refactor to have promises working on all layers.

The next step its to refactor the Provider Interface inside Core to adapt it to promises. 

```swift
public protocol CharacterProviding {
    func characters(offset: Int?) -> Promise<[Character], CharacterRepositoryError>
    func character(by id: Int) -> Promise<Character, CharacterRepositoryError>
}
```

Then the Provider Implementation inside our MarvelClient.

```swift
public func characters(offset: Int?) -> Promise<[Core.Character], CharacterRepositoryError> {
    service.characters(offset: offset)
        .then { $0.map(\.coreCharacter) }
        .catch { $0.repositoryError }
}

public func character(by id: Int) -> Promise<Core.Character, CharacterRepositoryError> {
    service.character(by: id)
        .then { try $0.first.map(\.coreCharacter) ?? CharacterRepositoryError.notFound }
        .catch { $0.repositoryError }
}
```

We got working now the Provider and the repository of core stops compiling, so let's fix the repository also.

```swift
public func characters(offset: Int?) -> Promise<[Character], CharacterRepositoryError> {
    provider.characters(offset: offset).then { result in
        completion(result.map { $0.run { $0.forEach { self.cache.set(value: $0) } } } )
    }
}
```

At this point, we see how we can make our code simpler again, to remove the `run` usage, if we extend the promises and add a new `then` that let us to do something with the Value, but not mutate it. This way, we operate the value, and then, return the promise again.
So, to end that method refactor, let's add the Promise extension inside the Promise framework.

```swift
func then(on queue: Queue = DispatchQueue.main,
          _ runValue: @escaping (T) -> Void) -> Promise<T, E> {
    map(
        on: queue,
        mapValue: {
            runValue($0)
            return Promise<T, E>(value: $0)
        },
        mapError: { Promise<T, E>(error: $0) }
    )
}
```

And using it, our character repository functions looks like:

```swift
public func characters(offset: Int?) -> Promise<[Character], CharacterRepositoryError> {
    provider.characters(offset: offset).then {
        $0.forEach { self.cache.set(value: $0) }
    }
}

public func character(with id: Int) -> Promise<Character, CharacterRepositoryError> {
    if let character = cache.get(id: id as Character.ID) as Character? {
        return Promise(value: character)
    } else {
        return provider.character(by: id).then { self.cache.set(value: $0) }
    }
}
```

For the last, we only need to update the two UseCases of the features.

```swift
extension GetCharacterDetail: GetCharacterDetailProtocol {
    func execute(with id: CharacterId) -> Promise<CharacterDetail, CharacterDetailError> {
        characterRepository.character(with: id)
            .then { CharacterDetail(with: $0) }
            .catch { CharacterDetailError(characterRepositoryError: $0) }
    }
}
```

And the GetCharacters: 

```swift
extension GetCharacters: GetCharactersProtocol {
    func execute(offset: Int?) -> Promise<[CharacterListModel], CharacterListError> {
        characterRepository.characters(offset: offset)
            .then { $0.map { CharacterListModel(with: $0) } }
            .catch { CharacterListError(characterRepositoryError: $0) }
    }
}
```

As we see until now, our maps are easier to read, and we've reduced a lot the code quantity.

Now time for our ViewModel and Presenter, this case it's a little bit different, to ilustrate that we can pass the functions to parameters that receives a closure. Because functions are named closures or a closure it's an anonimous function.

```swift
func loadCharacterDetail() {
    getCharacterDetail.execute(with: characterId)
        .then(getCharactersFinished)
        .catch(getCharactersFinished)
}
```


This is the actual code of the Presenter, as we see, now we need one more extension inside our Promises. Yes, the `.always` method that will execute the content when the promise finish. Also if it's fulfilled or rejected.
```swift
getCharacters.execute(offset: offset) { [weak self] result in
    guard let s = self else { return }
    s.loading = false
    switch result {
    case .success(let characters):
        s.getCharactersFinished(with: characters)
    case .failure(let error):
        s.getCharactersFinished(with: error)
    }
```

So let's develop the `.always`. Back to the Promises framework, let's create a new file named `Always`. This extension only have to do one thing, perform one block and then return the new promise with the received values.

```swift
@discardableResult
func always(on queue: Queue = DispatchQueue.main,
            _ do: @escaping () -> Void) -> Promise<T, E> {
    map(
        on: queue,
        mapValue: {
            `do`()
            return Promise(value: $0)
        },
        mapError: {
            `do`()
            return Promise(error: $0)
        }
    )
}
```

And using it on the CharacterListContainerPresenter.

```swift
func loadCharactersList() {
    guard !loading else { return }
    
    loading = true
    getCharacters.execute(offset: offset)
        .always { self.loading = false }
        .then(getCharactersFinished)
        .catch(getCharactersFinished)
}
```

That's it. We got now all our app working with our custom developed `Promises`. (commit: `d23d168`)

## Conclusions

- We saw how easy is to extend one `Package` adding new functionality while we don't touch the original code in our library.
- We performed a step by step refactor, by layers, covering how we can do this kind of recfactor in a real big app.
- We saw the power of promises, how them can make our code, simpler, more readable and robust.

Working with promises, it's a good first step to start with functional concepts like [Monads](https://en.wikipedia.org/wiki/Monad_(functional_programming)), the one it's a concept that we must fully understand, because it's the base of our code, Optionals are Monads, Arrays are monads, and like in this case the Promises.

Also we need to understand how the `map`, `flatmap`, and other functions works. This kind of functions allow us to remove `if`s and make our code more readable, because our data flow it's very clear, since we don't have two branches on the code (they are, but wrapped inside the result, promise or whatever), we only take one object, and map one value if need, and map the error if needed. And the mapping process it's even easier to read, because we know that one function give us one type, we take that type, mutate it, and return a new type. This is how all the core of this application it's working right now.


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