---
layout: post
title: Xcode at scale - Introducing to Promises - Creating our own framework
author: juli
---

This is the tenth step of a serie named **iOS at Scale** based on the next steps:
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

Since Swift it's here, the way of coding have changed. This is a Fact. On the old obj-c times are gone, on old times we only have Protocols and Clases, and we must build our code over that. We suffer a explosion of classes when doing architectures like VIPER, we was lead to build new classes wrapping other components, add extra layers of complexity, and in the end, write a lot of more code.

Generic and functions as a type bring us the option to start building very smalls components, with a very closed responsability, and start combining them.

Since creation, Swift it's a game changer.

We have new tools that may help us on our daily basics coding. **Our mindset have to change.** We still use the same principles, Solid, Kiss, OOP, and the most common design patterns are still here, but new friends are come too, we have Type Erasure, POP, Closures... And now that  it's providing his own RX framework like Combine, framework where Swift UI it's built, don't understand how these tools works is not an option.

We're going to make a little step inside functional programming, we're going to create a very little Promises framework, with a couple of clases, to lose the fear to this concepts, also, we're cover that we don't need a big external library like [PromiseKit](https://github.com/mxcl/PromiseKit), [Google Promises](https://github.com/google/promises), or even [RxSwift](https://github.com/ReactiveX/RxSwift) to get the benefits of FPP.

Lets start making!

Please go to `start-promises` on git (`git checkout start-promises`).

To generate the project with Tuist we must run `tuist generate` on the `Workspace.swift` folder. Check the a little bit the code and let's stop to think one minute.

The first step it's create our own `Package` as we are going to add only code, [SPM]() it's a great option, and we can share this package along our company, open source, what ever.

## Creating the SPM Package

As this package will not fit under `Modules` of our app because it really is a dependency, we will create another folder called `Libraries` to know that those packages are shared. Then on Library/Promises folder we execute `swift package init --type library`. We have our package ready. (commit: `ed82c10`)

Note: _like this package will be only one target and for iOS, I'll do a little cleanup_.
Note: _SPM allow us to share our packages to other platforms, following this kind of settups will make us able to reuse a lot of components_

I've removed the test for linux, and set that this library only build for iOS with minimum version iOS 10. Also moved the source code to `Sources` instead `Sources/Promises`. (commit: `ca06faa`)

Okey, we have the package, but we need set it as dependency and we able to work on it, more now that the package it's under initial development.

In order to do this, we need to add the package dependency to our `Support` Module, due that this Module it's available to the whole app, and we want `Promises` also available to the whole app.

Also, to have the SPM under development, we only have to add it to our Workspace ([doc](https://stackoverflow.com/questions/56627369/how-do-i-put-a-swift-package-in-edit-mode)). (tuist do this automatically when use the `path` Package initializer) (commit: `0af3a05`)

## Start to creating our Promises

Promises are objects that will help us doing asynchronous tasks, with fixed Types and allow us to mutating the given data by `then` & `catch` methods. Its very similar to the used method on the project `Done<Value, E: Error> = (Result<Value, Error>) -> Void` but with more syntax sugar, and easy to extend but not to modify (Open Close principle). 

The result approach it's fine, but learned the promises, learned a the basis of FPP.

Even using `Result` if we do this:

```swift
public typealias CharactersCompletion = (_ result: Result<[Character], CharacterRepositoryError>) -> Void
```

We're adding extra complexity, we are creating more `typealias` for each type of response on our `Repository` layer, and this is creating more code to maintain, less standard solutions... since we can have only one `Completion` declared on support:

```swift
public typealias Done<T, E: Error> = (Result<T, E>) -> Void
```

The first thing that help me a lot about FPP it's read a function with the following sentence, a function it's something that GIVEN some parameters, DO some work and RETURNS something. This is extreamly useful since we can have Objects, with function as properties, intializer parameters and combining this features with generics will make our code work easier.


Since our promises will execute some work, we need to add them the hability to choice where to perform that work. This will be expressed by a protocol called `Queue`. (commit: `fd69d7e`)

```swift
protocol Queue {
    func execute(_ work: @escaping () -> Void)
}
```

And our Promises will have an `State`, since they could be `pending`, `fullfilled` (completed) or `rejected` (completed with error).
So lets create an `enum State` with this possible states.
We end up with this: (commit: `6efbaa7`)

```swift 
enum State<Value, E: Error> {

    case pending()

    case fulfilled(value: Value)

    case rejected(error: E)


    var isPending: Bool {
        if case .pending = self {
            return true
        } else {
            return false
        }
    }
    
    var isFulfilled: Bool {
        if case .fulfilled = self {
            return true
        } else {
            return false
        }
    }
    
    var isRejected: Bool {
        if case .rejected = self {
            return true
        } else {
            return false
        }
    }

}

```

Now we're going to create the promise object, this object must be initializated with a work to perform, and a `Queue` where perform it. But we need a way to nofify the `Promise` when the work it's done. We're going to do this creating a new object called `Future`, this object has two blocks, one to _fullfill_ the `Promise`, and another to _reject_ it. (commit: `0ea74a8`)

```swift
public struct Future<T, E: Error> {
    
    public let fullfill: (T) -> Void
    public let reject: (E) -> Void
    
}
```

With the need of a place to store the callbacks that will be executed when the promise ends, a good choice it's to keep them on the `State.pending`, so we need to add a place that stores the work to perform when the promise completes. 

This work have two options fullfill and reject, since the promise can be completed with success or error. This fits perfectly with the Future object we've created, so let's add the Future into the `State.pending`, (because we only need to keep the blocks if the promise it's not completed)

We need that the pending state can be initialized with an empty future, so instead of a Future, let's add an Array of Futures. (commit: `1288f38`)

```swift
enum State<Value, E: Error> {

    case pending(future: [Future<Value, E>])

    case fulfilled(value: Value)

    case rejected(error: E)

}
```

Now that we have all those components, let start to creating our `Promise`. The first thing that we need it's an empty initialzer, and remember, `Promise` must has an `State`, and how we will mutate the promise, but we don't want that it changes his reference, it must be a class (final to be closed to modification) (commit: `0515e26`).

```swift
public final class Promise<T, E: Error> {
    
    private var state: State<T, E>
    
    public init() {
        self.state = .pending(futures: [])
    }
    
}
```

Now we must be able to create the promise with a work that when ends launch the fulfill or the reject method. So let's add it. (commit: `a7cfcc3`)

```swift
public final class Promise<T, E: Error> {
    
    private var state: State<T, E>
    
    public init() {
        self.state = .pending(futures: [])
    }
    
    public convenience init(on queue: Queue, _ work: @escaping (Future<T, E>) -> Void) {
        self.init()
        queue.execute {
            work(Future(fulfill: self.fulfill, reject: self.reject))
        }
    }

    private func fulfill(with value: T) {
        
    }

    private func reject(with error: E) {
        
    }

}
```

And we will need the option to create a `Promise` with Error or Value, so let's add those initializer methods. (commit: `06e592c`)

Now we are going to do the hard part. Lets make our promises work, how them work? Adding futures to the pending state. Thats all. Seriously. Thats all.

```swift 
private func addFuture(future: Future<T, E>) {
    switch state {
    case .pending(let futures):
        state = .pending(futures: futures + [future])
    case .fulfilled(let value):
        future.fulfill(value)
    case .rejected(let error):
        future.reject(error)
    }
}
```

That it's the most important method on the `Promise`. Let's explain all steps on that method to be clear. If the promise is pending (work it's currently being executed), we take the prevous futures and add the new one, but we keep `.pending`.
If the promise has done the work, and there is a value, we execute the given future.fulfill passing to it the value. 
And the last, if has done the work with an error, we call the rejected with the given error. (commit: `981cd16`)


We still have empty functions on `Promise.fulfill` and `Promise.reject` so lets fill them.

```swift
private func fulfill(with value: T) {
    guard case .pending(let futures) = state else { return }
    state = .fulfilled(value: value)
    futures.forEach { $0.fulfill(value) }
}

private func reject(with error: E) {
    guard case .pending(let futures) = state else { return }
    state = .rejected(error: error)
    futures.forEach { $0.reject(error) }
}
```

This methods can be only called one time, the thime that the work ends, then they mutate the state to `fulfilled` or `rejected`, and then the rest of `thens` and `catch` will be instantly executed because the promise has done the job. (commit: `e858fd4`)

Now that we understand how a promise works, we need to make the state mutation thread safe. In order to do this, the only thing we need it's a private queue and make the state mutations inside that queue.

Finally Promise looks like: (commit: `d660548`)

```swift
public final class Promise<T, E: Error> {
    
    private var state: State<T, E>
    private let lockQueue = DispatchQueue(label: "promise.lock.queue", qos: .userInitiated)
    
    public init() {
        self.state = .pending(futures: [])
    }
    
    public init(value: T) {
        self.state = .fulfilled(value: value)
    }
    
    public init(error: E) {
        self.state = .rejected(error: error)
    }
    
    public convenience init(on queue: Queue, _ work: @escaping (Future<T, E>) -> Void) {
        self.init()
        queue.execute {
            work(Future(fulfill: self.fulfill, reject: self.reject))
        }
    }

    private func fulfill(with value: T) {
        lockQueue.async {
            guard case .pending(let futures) = self.state else { return }
            self.state = .fulfilled(value: value)
            futures.forEach { $0.fulfill(value) }
        }
    }

    private func reject(with error: E) {
        lockQueue.async {
            guard case .pending(let futures) = self.state else { return }
            self.state = .rejected(error: error)
            futures.forEach { $0.reject(error) }
        }
    }
    
    private func addFuture(future: Future<T, E>) {
        lockQueue.async {
            switch self.state {
            case .pending(let futures):
                self.state = .pending(futures: futures + [future])
            case .fulfilled(let value):
                future.fulfill(value)
            case .rejected(let error):
                future.reject(error)
            }
        }
    }
    
}
```

At this point that we're going to start using the Queue protocol, we're going to extend `DisptchQueue` to conform the protocol and add it as default parameter on the `Promise.init(queue, work)`.

```swift
extension DispatchQueue: Queue {
    func execute(_ work: @escaping () -> Void) {
        async { work() }
    }
}
```

```swift
public convenience init(on queue: Queue = DispatchQueue.global(qos: .userInitiated), _ work: @escaping (Future<T, E>) -> Void) {
        self.init()
        queue.execute {
            work(Future(fulfill: self.fulfill, reject: self.reject))
        }
    }
```

Now we are going to add the `then` method, this method will set (or call if finished) fulfill and reject methods. (commit: `a7279a0`)

```swift
@discardableResult
public func then(on queue: Queue = DispatchQueue.main, _ fulfill: @escaping (Value) -> Void, _ reject: @escaping (Error) -> Void) -> Promise<T, E> {
    addFuture(future: Future(fulfill: fulfill, reject: reject))
    return self
}
```

But as we can see, `then` method expects to execute on an specified queue. This means that we need to update our `Future` object to perform his clousures on that queue. 

```swift
public struct Future<T, E: Error> {
    
    public typealias Fulfill = (T) -> Void
    public typealias Reject = (E) -> Void
    
    public let fulfill: Fulfill
    public let reject: Reject
    private let queue: Queue?
    
    init(on queue: Queue? = nil, fulfill: @escaping Fulfill, reject: @escaping Reject) {
        self.queue = queue
        self.fulfill = fulfill
        self.reject = reject
    }
    
    func callFulfill(value: T, _ completion: @escaping () -> Void = {}) {
        assert(queue != nil, "callFulfill only can be called if queue is not nil")
        queue!.execute {
            self.fulfill(value)
            completion()
        }
    }
    
    func callReject(error: E, _ completion: @escaping () -> Void = {}) {
        assert(queue != nil, "callReject only can be called if queue is not nil")
        queue!.execute {
            self.reject(error)
            completion()
        }
    }
    
}
```

To avoid this complex `Future` struct and merge two functionalities on the same object, (pass to the user to complete the promise, and add it as callback to the pending state) we're going to create a new struct `Callback` to handle this and let the Future only to the user.

You can check the end of this changes on `fe12512`.

Now we must add a function that mutate both values inside our promise, the error to a new error and the Value to a new Value. 
This function, adds a `Callback` to our `Promise`, that will execute the given blocks on the given queue when the promise ends with the promise result, and create a new `Promise` with the result of that `Callbacks`. So the mutation happens inside the maps.

Like we are typing the error, our mapValue if `throws` must `throw` always the new promise type error, and the mapp error is needed for mutate from the first `Promise` error to the new `Promise` error (commit: `0a257c5`).

Also I've marked as private the old `then`. (I'll why explain a little bit later). 

```swift
@discardableResult
public func map<NewValue, NewError: Error>(on queue: Queue = DispatchQueue.main,
                                            _ mapValue: @escaping (T) throws -> Promise<NewValue, NewError>,
                                            _ mapError: @escaping (E) -> NewError) -> Promise<NewValue, NewError> {
    Promise<NewValue, NewError> { future in
        self.then(
            on: queue,
            fulfill: { oldValue in
                do {
                    try mapValue(oldValue).then(on: queue, fulfill: future.fulfill, reject: future.reject)
                } catch let error {
                    future.reject(error as! NewError)
                }
            },
            reject: { oldError in
                future.reject(mapError(oldError))
            }
        )
    }
}
```

With that map done, to map Values and Errors, now we can create the `then` to map only the value. This function it's pretty easy. We only need to use the last function, but only updates de `Value` type. (commit: `10731ba`)

```swift
@discardableResult
public func then<NewValue>(on queue: Queue = DispatchQueue.main,
                            _ mapValue: @escaping (T) throws -> NewValue) -> Promise<NewValue, E> {
    map(
        on: queue,
        mapValue: { Promise<NewValue, E>(value: try mapValue($0)) },
        mapError: { Promise<NewValue, E>(error: $0) }
    )
}
```

And then, the same but for errors. (commit: `92e8a7d`)
```swift
@discardableResult
public func `catch`<NewError: Error>(on queue: Queue = DispatchQueue.main,
                                        _ mapError: @escaping (E) -> NewError) -> Promise<T, NewError> {
    map(
        on: queue,
        mapValue: { Promise<T, NewError>(value: $0) },
        mapError: { Promise<T, NewError>(error: mapError($0)) }
    )
}
```

Once we have all this functions inside the Promise, it's time to check that it works, and to do this, we can simply add a couple of tests.

At this time, I was testing the `.catch` and to only catch without mutate the Error type, I've needed to extend the Promise and add a new catch method. 

Also I've done a little refactor, creating two new files, `Then.swift` and `Catch.swift` where I extracted those method, to take some code out of how the promise works, due that really those methods are syntactic suggar for what we already have.

Finally, you can go to this commit and check the test passing (commit: `166f7bb`).

## Conclusion

We created from the scratch a very simple `Promise` framework that allow us to mutate values and perform asynchronous task. 

We combined `enums` with `structs`, we worked with functional concepts like [Monads](https://khanlou.com/2015/09/what-the-heck-is-a-monad/).

If we understand how this `Promise` works, we will understand easier new concepts like subscribers, new [SwiftUI](https://developer.apple.com/xcode/swiftui/) `@ObservedObjects`, swift `Combine`, and we can adopt a less code way, with less options to write bugs.

Notice that this Promise Framework its highly inspided on [Khanlou](https://github.com/khanlou/Promise) Promise frameworks, but allowing us to type the Error.

Typing the error limit us on what erros can be thrown, and since Swift doesn't allow us to type errors under throwing functions, if we throw a not expected error type inside map or then, these promises **will crash on runtime**. But typing erros we will have the option of `switch` over a known error.

To avoid this, you can do remove the error typed on the promise, or use one of the most popular frameworks named on the start of the article that doesn't allow to type the promise.

I encourage you to test the framework, start to extend it and understand how it works closely. Maybe not promises, but FPP combined with OOP like nowadays we can do, it's the future of how the apps will be made, and we need know it.
