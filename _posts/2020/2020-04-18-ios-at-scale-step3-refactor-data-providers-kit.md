---
layout: post
title: Xcode at scale - Refactor Data Providers - applying inversion of control
author: juli
---

This is the fourth step of a serie named **iOS at Scale** based on the next steps:
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


On the previous post we've seen how a simple `Package` could make our life easier and help us to see parts of our code highly coupled. A good excercise to see if our coude it's coupled (along make unit testing to see if we can mock without effort our dependencies) is to think, what should I do if I wan't to make from this function a framework? Some times our code could look fine, but we can still have highly coupled things passing unnoticed. 

For this chapter, the first thing we're going to do it's make our app compile again so...

Go to the `step-2` tag.

## Start refactor DataProvidersKit

The first class to refactor will be `CharacterService` since other classes are depending from them.

```swift
protocol CharacterServiceProtocol: AnyObject {
    func characters(nameStartsWith: String?,
                    offset: Int?,
                    completion: @escaping CharactersCompletion)
    func character(with id: Int, completion: @escaping CharacterCompletion)
}
```

This kind of names could be avoid if we follow  guidelines, so let's change the name to, `CharacterServicing`. (commit: `5396064`)

Also, we're going to remove this extra complexity typealiases: (commit: `7f86c80`)

```swift
public typealias CharacterCompletion = (_ result: Result<Character, CharacterRepositoryError>) -> Void
public typealias CharactersCompletion = (_ result: Result<[Character], CharacterRepositoryError>) -> Void
```

To finally use the `Done<T, E>` provided by `Support` on the previous chapter. This way we see on all methods whats happening and don't hide info under aliases, that are on other files.

```swift
protocol CharacterServicing: AnyObject {
    func characters(nameStartsWith: String?, offset: Int?, completion: @escaping Done<[Character], CharacterRepositoryError>)
    func character(with id: Int, completion: @escaping Done<Character, CharacterRepositoryError>)
}
```

Not it's time for the `Service` itself. And we're starting from:

```swift
class CharacterService {
    let apiClient: MarvelAPIClient
    
    init(apiClient: MarvelAPIClient) {
        self.apiClient = apiClient
    }
}

extension CharacterService: CharacterServicing {
    func characters(nameStartsWith: String?,
                    offset: Int?,
                    completion: @escaping Done<[Character], CharacterRepositoryError>) {
        
        let request = GetCharacters(name: nil,
                                    nameStartsWith: nameStartsWith,
                                    offset: offset)
        apiClient.send(request) { response in
            switch response {
            case .success(let dataContainer):
                let networkCharacters = dataContainer.results
                guard networkCharacters.count != 0 else {
                    completion(.failure(.notFound))
                    return
                }
                
                let characters = networkCharacters.map { Character(withResponse: $0)}
                
                completion(.success(characters))
            case .failure(let error):
                completion(.failure(CharacterRepositoryError(withResponseError: error)))
            }
        }
    }
    
    func character(with id: Int, completion: @escaping Done<Character, CharacterRepositoryError>) {
        let request = GetCharacter(characterId: id)
        apiClient.send(request) { response in
            switch response {
            case .success(let dataContainer):
                let networkCharacters = dataContainer.results
                guard let networkCharacter = networkCharacters.first else {
                    completion(.failure(.notFound))
                    return
                }
                let character = Character(withResponse: networkCharacter)
                
                completion(.success(character))
            case .failure(let error):
                completion(.failure(CharacterRepositoryError(withResponseError: error)))
            }
        }
    }
}
```

As we see, `nameStartsWith` isn't used, so we're going to remove it. Also, like `CharacterService` isn't a class to make subclasses of it, we're going to mark it like `final`. 

_Adding final to your method it means more than just a compile time error. It will internally optimise your code, it will generate a more efficient representation, furthermore it can result into a faster execution. Next time that you create a method and you are not expecting to be overridden remember to mark as final._ ([Ref](https://medium.com/@MarcioK/swift-final-functions-under-the-hood-2deccd0b9437))

At this point, we see our `characters` method a little bit simpler. (commit: `5e3f0d0`)

```swift
func characters(offset: Int?, completion: @escaping Done<[Character], CharacterRepositoryError>) {
        provider.characters { result in
            switch result {
            case .success(let dataContainer):
                let networkCharacters = dataContainer.results
                guard networkCharacters.count != 0 else {
                    completion(.failure(.notFound))
                    return
                }
                
                let characters = networkCharacters.map { Character(withResponse: $0)}
                
                completion(.success(characters))
            case .failure(let error):
                completion(.failure(CharacterRepositoryError(withResponseError: error)))
            }
        }
        
    }
```

Wee need to add the offset value to our provider on `MarvelAPI` (commit: `dd21dfc`)

```swift
public func characters(offset: Int?, _ done: @escaping (Result<[Character], MarvelError>) -> Void) {
    client.perform(Endpoint(path: "/v1/public/characters", parameters: ["offset": offset ?? 0])) { (result: Result<Page<[Character]>, Error<MarvelError>>) in
        done(result.map(\.results).mapTerror(\.marvelError))
    }
}
```

And then, lets adapt to this `CharacterService`. Like in the previous post, we need an extension of `MarvelError` to map it easily to CharacterRepositoryError.

```swift
extension MarvelError {
    var repositoryError: CharacterRepositoryError {
        switch self {
        case .server(let code, let message): return .marvelError(code: code, message: message)
        case .underlying(let error): return .unknow(error)
        }
    }
}
```

And after that, we only need to map a couple of things. At the first time, this could be a little confusing, but lets explain what's happening inside.
- We receive `Result<[MarvelClient.Character], MarvelError>`
- Pass it to the completion, but mutated, look at the chain. `result.map { }.map { }`. Each map take the previous value and returns a new value.
- The first map will take that result and transform it to `Result<[DataProvidersKit.Character], MarvelError>`
- The second map will take the first maping result `Result<[DataProvidersKit.Character], MarvelError>` and mutate it to `Result<[DataProvidersKit.Character], CharacterRepositoryError>`

As wee see, one map mutates the `.success.` branch of the result and the other one mutates the `.failure` branch.

(commit: `dabee48`)

Note: _I'm allowing empty array results_

```swift
func characters(offset: Int?, completion: @escaping Done<[Character], CharacterRepositoryError>) {
    provider.characters(offset: offset) { result in
        completion(result
            .map { $0.map { Character(withResponse: $0) } }
            .mapTerror(\.repositoryError)
        )
    }
}
```

Now, lets do the single character method.
- We receive `Result<[MarvelClient.Character], MarvelError>`
- The first map will take that result and transforming it taking only the first value it `Result<Character, MarvelError>`
- The second map will take the first maping result `Result<Character, MarvelError>` and mutate it to `Result<[DataProvidersKit.Character], CharacterRepositoryError>`

```swift
provider.character(by: id) { result in
    completion(result
        .map { $0.first.map { Character(withResponse: $0) } ?? "Error" }
        .mapTerror(\.repositoryError)
    )
}
```

But as we can see, what if our response has an Empty array, as you can see above, we need to throw an error, but Result `map` doesn't allow us to do that, so let's add one more function to our previous extension on support.

```swift
func tryMap<NewSuccess, NewFailure>(transformSuccess: (Success) throws -> NewSuccess,
                                    transformFailure: (Failure) -> NewFailure) -> Result<NewSuccess, NewFailure> {
    do {
        switch self {
        case .success(let success): return .success(try transformSuccess(success))
        case .failure(let error): return .failure(transformFailure(error))
        }
    } catch {
        return .failure(error as! NewFailure)
    }
}
```

And an `Optional` extension to throw errors like `?? error`, _I really like this extension_. (commit: `ed9d6dd`)

```swift
public extension Optional {
    static func ?? (optional: Self, error: Error) throws -> Wrapped {
        switch optional {
        case .some(let value): return value
        case .none: throw error
        }
    }
}
```

And our final result of `character(by: "id")` it's: (commit: `584ba51`)

```swift
func character(with id: Int, completion: @escaping Done<Character, CharacterRepositoryError>) {
    provider.character(by: id) { result in
        completion(result.tryMap(
            transformSuccess: { try $0.first.map { Character(withResponse: $0) } ?? CharacterRepositoryError.notFound },
            transformFailure: { $0.repositoryError })
        )
    }
}
```

And the last step, fix the `Assemblies` (commit: `c7909b2`).

And then, lest fixt the dependency graph and the parameter `startsWith` from all layers. (commit: `8112897`)

Now our app it's compiling again. Let's rename `DataProvidersKit`

## Renaming

Since we're using [Tuist](www.tuist.io), rename a Module, is quite simple. Let's change the folder name, the name inside the `Project.swift`, and then update dependencies. (commit: `23680f2`)

## Aplying Inversion Of Control

Inversion of Control it's a way to avoid coupling our `Core` to our Modules/Libraries, instead of that, our modules will depend on our `Core`, this way, will be the `Core` the responsible of expose those protocols that the other libraries must conform. Giving us the ability to replace those libraries with painless, due that our application code doesn't need to change anything.

In order to do this, we're going to change the dependency graph. Actually it's `Features` -> `Core` -> `MarvelClient`, and it will be `Features` -> `MarvelClient` -> `Core`, but really, our features won't use code under `MarvelClient`, it will use only `MarvelClient` to take the `Core`.

So again, let's edit the `Project.swift` of those libraries. (commit: `dace026`)

Now, we need to move the `CharacterProviding` inside the `Core` Module. (commit: `fc5a1e5`) Then remove the dependencies to `MarvelClient`from core. (commit: `fd8f8eb`)

Models mapping must be moved from MarvelClient to Core, must be done inside MarvelClient. (commit: `67afea5`)

The last, find & replace from old `DataProvidersKit` to the new `Core` (commit: `9a7af3e`), as you can see, without touching anything of the code inside the Features or AppCore, we keep compiling the app, and we've gained a lot of flexibility to change how the network client works or how the `MarvelClient` works.


## Conclusions

At this post finally we've some key points about how modern techniques could **reduce dramastically** the code quantity and give us more code quality. Not only that, now we are more flexibles to change, and with the possibility of share our HTTPClient with the world if we want to.

Also, we've gained some functionality. Before we haven't the ability of read the Error body in our code and be able to do that it's something powerful. How many times we don't need to know the http code and some parameter in the body to do one action or other? This way we can archieve that.

In the middle way, we are **killing** some bad Modules responsabilities, excesive classes and nested ifs, making the data flow easier to read than before and harder to introduce bugs. Each layer do a minimal task, take some data, map those data. That's the most complex layer, and if something fails, we're mapping the `Error` with known Types. Without forget the easiness to make test, test-doubles and fake dependencies

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