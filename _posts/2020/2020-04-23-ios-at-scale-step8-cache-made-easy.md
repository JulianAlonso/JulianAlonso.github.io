---
layout: post
title: Xcode at scale - Cache made easy
author: juli
---

This is the nineth step of a serie named **iOS at Scale** based on the next steps:
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

Yes, we're near to the end. We're on Extra-Balls.

For this chapter, I want to build a simple `Cache` with protocol oriented programming. This way, I'll cover how to develop a new component, with a new package and how to merge it inside our stack. And we will find out how easy it is.

The first thing I'm going to do, again, it's to create it with the option to share. For this, like on previos post, I'm going to use [SPM](https://swift.org/package-manager/).

So, let's work! Go to the `step-9` on git.

## Creating a cache, POP

The first thing we need it's to create the new folder under `Libraries`, at the moment, this library will only contain a cache, but like this could scale, let's name it `Storage`. And like before, a little clean up into the generated library. (commit: `1a91ca7`)

Now, that we have our package created, let's add it as dependency to our `Core` and start to work on it. (commit: `413c7df`)

I've also changed how these packages under development are managed on tuist. `Package` on [Tuist](tuist.io) can be defined with `path` instead `url`, this will insert them on our project directly without need to add them to the `Workspace` as extra files (as I do). But, to ilustrate better what libraries are under development, I'll keep having them as extra files. This way, we have a folder `Libraries` available on our workspace.

To create the `Cache`, I'm going to define the protocol with the functions that I want to use. This way, we're out of how them will be implemented, and we'll keep the usage as easy as possible.

The protocol will be named `Caching` and with two functions, to `set` a value and to `retrieve` a value.

```swift
public protocol Caching {
    func set(key: String, value: Any)
    func get(key: String) -> Any?
}
```

This could be our first `Cache` protocol to use, but nowadays that we have `generics`, let's improve the value `Type`.

```swift
public protocol Caching {
    func set<T>(key: String, value: T)
    func get<T>(key: String) -> T?
}
```

And now, we have the main interface of our `Caching` object.

Let`s do the implementation.

```swift
public final class MemoryCache {
    private let cache = NSCache<String, Any>()
}
```

Like `NSCache`, need a class type to be the key of the dictionary, let's create a new object called `Key`, and how it needs `Any` to be a class, let's create a wrapper for the value, called `Value`.

```swift
final class Key: Hashable {
    
    private let key: String
    
    init(_ key: String) {
        self.key = key
    }
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(key)
    }
    
    static func == (lhs: Key, rhs: Key) -> Bool {
        lhs.key == rhs.key
    }
}


final class Value {
    let value: Any
    
    init(_ value: Any) {
        self.value = value
    }
}
```

And now to make it work, the `Key` must be an `NSObject` with `hashvalue` and `isEqual`, so, to do this, I'm going to create a `WrappedKey`, that wraps our real `Key`. And I'll add an extension to `Key` to create an `WrappedKey` fast, easy and located.

```swift
extension MemoryCache {
    final class WrappedKey: NSObject {
        let key: Key
        
        init(_ key: Key) {
            self.key = key
        }
        
        override var hash: Int { key.hashValue }
        
        override func isEqual(_ object: Any?) -> Bool {
            guard let wrapped = object as? WrappedKey else { return false }
            return self.key == wrapped.key
        }
    }
}

private extension Key {
    var wrapped: WrappedKey { .init(self) }
}
```

```swift
public final class MemoryCache {
    private let cache = NSCache<WrappedKey, Value>()
}
```

Now, let's create the implementation of `Caching`. (commit: `c4cd3c3`)

```swift
extension MemoryCache: Caching {
    public func set<T>(key: String, value: T) {
        cache.setObject(Value(value), forKey: Key(key).wrapped)
    }
    
    public func get<T>(key: String) -> T? {
        cache.object(forKey: Key(key).wrapped)?.value as? T
    }
}
```

We got working now our cache. But to make it easier to use, we're going to extend the protocol and add some util functions. Like we got the main two methods working, we're going to create some methods that will handle the key for us and make our life easier.

```swift
func set<T>(value: T) {
        
}
```

I want to use that interface, and use an `ID` as key for the object, so, we need that `T` conforms `Identifiable`.

Like Identifiable it's only available for iOS 13 or more, let's add inside `Cache` our `Identifiable` implementation.
If we go to the code on the `Identifiable` protocol, we will find out that the code it's really simple. So copy & paste the code inside a new `Cache` file. This way, we're working with our protocol, but at the moment that we update the project version to iOS 13, removing this implementation will make the project works without change anything more.

```swift
public protocol Identifiable {

    /// A type representing the stable identity of the entity associated with `self`.
    associatedtype ID : Hashable

    /// The stable identity of the entity associated with `self`.
    var id: Self.ID { get }
}
```

```swift
func set<T>(value: T) where T: Identifiable {
        
}
```

And now let's make the implementation.

```swift
func set<T>(value: T) where T: Identifiable {
    set(key: "\(value.id)", value: value)
}

func get<T>(id: T.ID) -> T? where T: Identifiable {
    get(key: "\(id)")
}
```

We're finding ourselves creating the `Key` each time we want use it, so this time we can create an extension of `Key` to be initialized with an `ID` of an `Identifiable` object.

```swift
extension Key {
    convenience init<T>(id: T.ID) where T: Identifiable {
        self.init("\(id)")
    }
}
```

And our Caching implementation looks: 

```swift
public extension Caching {
    func set<T>(value: T) where T: Identifiable {
        set(key: Key(id: value.id), value: value)
    }
    
    func get<T>(id: T.ID) -> T? where T: Identifiable {
        get(key: Key(id: id))
    }
}
```

Now that we have an object with an `ID`, we got the `Key`, but what happens if we want to store two different kind of object, wich ID it's the same? We will find us overriding a previous value that we dont want to override. So, how we got the type when saving the object and when retrieving. Let's combine the `Type` with the `ID`. The responsible of this work it's `Key` initializer itself, so we only need to work on it.

```swift
private extension Key {
    convenience init<T>(id: T.ID, class: T.Type) where T: Identifiable {
        self.init("\(type(of: T.self))\(id)")
    }
}
```

Now, let's update the `Caching` interface, to receive a `Key` instad of `String`. And this is how finally look our `Caching` protocol.  (commit: `4e63906`)

```swift
public protocol Caching {
    func set<T>(key: Key, value: T)
    func get<T>(key: Key) -> T?
}

public extension Caching {
    func set<T>(value: T) where T: Identifiable {
        set(key: Key(id: value.id, class: T.self), value: value)
    }
    
    func get<T>(id: T.ID) -> T? where T: Identifiable {
        get(key: Key(id: id, class: T.self))
    }
}
```

## Using the Cache from the Core

Now, we got the `Cache` available from `Core`. Let's use it on our repository.

Note: _I'm going to couple `Core` to our `Storage` library, but we can create our own `Caching` protocol inside core, and wrap the  `Storage.Caching` with it and stay decoupled (that interface only has to forward the get, set functions to the Storage.Cache functions)._

Finally, let's update our Repostiory to have the `Caching` dependency. (commit: `8c78256`)

```swift
public final class InternalCharacterRepository {
    let provider: CharacterProviding
    let cache: Caching
    
    public init(provider: CharacterProviding, cache: Caching) {
        self.provider = provider
        self.cache = cache
    }
}
```

And now, let's create a component inside app to register `Caching` and update Repostiory factory to inject it. (commit: `71a88b1`)

```swift
let storageComponent = Component {
    single { MemoryCache() as Caching }
}

let repositoryComponent = Component {
    factory { InternalCharacterRepository(provider: $0(), cache: $0()) as CharacterRepository }
}
```

Finally, lets make our Repository cache the characters.

```swift
public func characters(offset: Int?, completion: @escaping Done<[Character], CharacterRepositoryError>) {
    provider.characters(offset: offset) { result in
        result.map { $0.forEach { self.cache.set(value: $0) } }
    }
}
```

As we see, we need `Character` to conform `Identifiable`. So let's do it.
```swift
extension Character: Identifiable {}
```

And I've introduced one more thing. A protocol named `Runnable` with the only idea of do more things in one line. 

```swift
public protocol Runnable {}

public extension Runnable {
    @inlinable
    func run(_ work: (Self) -> Void) -> Self {
        work(self)
        return self
    }
}
```

As we see, the `Runnable` only takes itself, pass it to a closure to do some work, and then return itself. This way, we can use the `map` of the `result` ot iterate the objects instead of mutate them. Because like we're inside the completion block, we need the map to return something that will be what be passed to the completion parameter.
And like the `map` only get's called if our `result` it's `success`, we get the cache working only on the `success` branch. (commit: `47ac463`)

```swift
public func characters(offset: Int?, completion: @escaping Done<[Character], CharacterRepositoryError>) {
    provider.characters(offset: offset) { result in
        completion(result.map { $0.run { $0.forEach { self.cache.set(value: $0) } } } )
    }
}
```

Now, we're storing our objects but not reading them. Let's make our repostiory read for the character detail.

```swift
public func character(with id: Int, completion: @escaping Done<Character, CharacterRepositoryError>) {
    if let character = cache.get(id: id as Character.ID) as Character? {
        completion(.success(character))
    } else {
        provider.character(by: id, completion)
    }
}
```

And for the last, if we don't have cached the character, we make the request, but then we're not storing that character on the cache, so let's make our `character(by: id)` stores also the response. (commit: `4d26f9f`)

```swift
public func character(with id: Int, completion: @escaping Done<Character, CharacterRepositoryError>) {
    if let character = cache.get(id: id as Character.ID) as Character? {
        completion(.success(character))
    } else {
        provider.character(by: id) {
            completion($0.map { $0.run { self.cache.set(value: $0) } })
        }
    }
}
```

We got our cache working!

## Conclusion

This chapter we've seen how to do a very, very easy cache. Using [Protocol Oriented Programmin]() to with only a couple of methods, handle how new Identifable Items are saved and retrieved. Also covering the use of the NSCache and how we can wrap old needs like extends from NSObject in our Swift classes.

We've also covered how to develop and integrate a new library on our code base it's extramly easy and painless without the needs of update any of our Features modules.

Also we've seen how `map` combined with the new protocol `Runnable` allow us to do more with less, and how combinning these little functions we end up building big features.

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