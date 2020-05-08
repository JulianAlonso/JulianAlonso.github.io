---
layout: post
title: Copying objects in Swift
author: juli
---

How many times, working with `struct`s, we want to grab a copy, mutate one value, and then return the mutated value. The traditional way of do this, always has been the next lines of code:

```swift
var copy = whatever
copy.foo = bar
return copy
```

With those lines, we can make mistakes, we can return `whatever` instead of `copy`, ending with the wrong value. Also make our coder harder to read, because we're writting three lines to perform a single operation, an operation not named, and we need to see what variable it's changing to know what's happening.

But nowadays, we got some protocols and features on the language that could help us to avoid that.

Not only that, we can make our code **more readable**, and **self explaining**. We're going to introduce a protocol that will help us to do this. 

For our goal, we need to have the ability of use `object.copy { $0.foo = bar }`. This function it's a pure function, it's a function designed inside a protocol, so all our objets will have it only implementing the protocol.

Kotlin has this feature included on the language, let's see an example:

```kotlin
data class User(val name: String = "", val age: Int = 0)

val jack = User(name = "Jack", age = 1)
val olderJack = jack.copy(age = 2)
```

This `copy` funciton is really useful when working with `map`s or `flatMap`s, or any type that let's us map over it. Like `Promises`, `Observables`, `Result`... We can map the value inside the wrapper, and then return it mutated, in one line. Our code will be easier to read and follow, but also more expressive and more concise. But this is not only useful when working with Wrapping types, we can even use this, for example, tag an item as favourite, we take the value from the DB, mutate the copy and then save the mutated copy, avoiding to have mutating objects on our code.

We can take the given data class, mutate it, and return the new, **in one line**. To give you more context, if we have the next user `struct`, and we want to do a function, to increment x years of the user age, I need to implement the next function:

```swift
struct User {
    let name: String
    let age: Int

    func older(by years: Int) -> User {
        //this doesn't work
        copy(age = age + years)
    }
}

```

Like swift doesn't have this function included on the language, we can do the next:

```swift
struct User {
    let name: String
    let age: Int

    func older(by years: Int) -> User {
        copy(age: age + years)
    }
}

extension User {
    func copy(name: String? = nil, age: Int? = nil) -> User {
        User(name: self.name ?? name, age: self.age ?? age)
    }
}
```

The main key point of this, is that we are mutating the value, but as you can see, we don't have the function as `mutating`. Because the value that we mutate it's a copy instead of the original object. This way we can avoid side effects, caused by mutate an object that it's inside another object. We should try to make as many pure functions as we can. We avoid mutate things without know what we're doing and these functions are easier to test and easier to read since we don't need to go out of the scope of the function to see whats happening...

Back to the last example, we have the same function as kotlin has and the `older` function working, but we have to write a lot of boilerplate code, each time we want have it available inside one object. We could use [sourcery](https://github.com/krzysztofzablocki/Sourcery) to generate this kind of functions automatically, but this time, I want to make a protocol, with the goal of only implementing it, we can do this kind of operation over any `struct`.

To do this, we're going to work with the next protocol.

```swift
protocol Copyable {}
```

And then, let's add some functions to mutate a fresh copy and then return it mutated.

```swift
extension Copyable {
    func copy(_ mutator: (inout Self) -> Void) -> Self {
        var copy = self
        mutator(&copy)
        return copy
    }
}
```

This is how we can mutate `self` with the given closure and then return the mutated copy of self. But we must pay a price, the price of having our properties inside the object as `var` instead `let`.  We need to make our objects variables as vars to be allowed to mutate them. Our `User` with this looks like

```swift
struct User {
    let name: String
    private(set) var age: Int

    func older(by age: Int) -> User {
        copy { $0.age = $0.age + 1 }
    }
}
```

Now, starting from the basic `copy` function, we can add more syntactic sugar to make it more readable. How? Using Keypaths in our `Copyable` protocol and using inside the old `copy` function.

```swift
extension Copyable {
    func replacing<T>(to keyPath: WritableKeyPath<Self, T>, value:  T) -> Self {
        copy { $0[keyPath: keyPath] = value }
    }
}
```

In our `User` code: 

```swift
struct User {
    let name: String
    private(set) var age: Int

    func older(by years: Int) -> User {
        replacing(\.age, to: age + years)
    }
}
```

This kind of functions and way of programming, lead us to make our code more readable, while keeping non mutating objects. 

We can even extend more the `Copyable` extension with more functions to have for example an `adding` function that will look like:

```swift
func adding<T>(to keyPath: WritableKeyPath<Self, T>, value: T, merging: (T, T) -> T) -> Self {
    copy { $0[keyPath: keyPath] = merging($0[keyPath: keyPath], value) }
}
```

Now, using the new function, our code under `User` will look like: 

```swift
struct User: Copyable {
    let name: String
    private(set) var age: Int
    
    func older(by years: Int) -> User {
        adding(to: \.age, value: years, merging: +)
    }
}
```

Like `+` it's a function that takes two values with the same type and returns the new one, we don't need to create the closure and add the values inside, the clousure will be the add function itself.

The last trick is the one with the highest price. If we make our structs properties vars, without `private(set)`, we could use the `adding` or `replacing` functions outside the struct object. The price will be the ability to mutate the properties without using our protocol functions always that our object be mutable. This could lead us to have some problems (that some times we can fix).

Let's supose we have the next ViewModel with an `State`:

```swift
final class ViewModel {
    var state: State
} 
```

Each time we want to update the state properties, if it is an struct instead an enum, after loading some data, we could do:

```swift
state.data = loadedData
```

On the other hand, if State is inside a parent class, we could close the state modification only by function

```swift
open class ViewModel<State> {
    private(set) state: State

    final func update(state: State) {
        self.state = state
    }
}
```

Or even:

```swift
open class ViewModel<State: Copyable> {
    private(set) state: State

    final func update(state mutator: (inout State) -> Void) {
        self.state = state.copy(mutator)
    }
}
```

This way, update the state after get some data should be:

```swift
update(state: { $0.data = someData })
```

We can even go further with functional.

## Extra tip.

Like you'll see, this is going to be less usable than the copy function, but we're here only to enjoy.

In swift, we have the option working with types that given a type returns a function, a funtion that given the type instance, return itself.
Let's see the example with the User.

We can write `User.older`, and this will return a function that given the user, return a the `older` function itself (`(User) -> ((by: Int) -> User)`). 

Knowing that, we can update our `update(state: )` to something that receives that kind of functions and the value. So we can endup with the following syntax in our ViewModel and remove the Copyable State requirement. 

```swift
final class UserVM: ViewModel<User> {
    
    func olderUser() {
        update(state: User.older, by: 11)
    }
    
}
```

The function on the ViewModel, will receive a function that given the State and the value, returns a function that given a value, returns the State.

```swift
class VM<State> {
    private(set) var state: State {
        didSet { print(state) }
    }
    
    init(state: State) {
        self.state = state
    }
    
    func update<T>(state mutator: (State) -> ((T) -> State), by: T) {
        self.state = mutator(state)(by)
    }
    
}
```