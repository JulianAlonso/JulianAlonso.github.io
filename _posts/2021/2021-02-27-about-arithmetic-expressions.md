---
layout: post
title: About Arithmetic Expressions
author: juli
---

Some days ago I watched this [episode](https://www.pointfree.co/episodes/ep26-domain-specific-languages-part-1) from PointFree , and I wanted to come over with a couple of exercises playing with the arithmetic expression of the exercise.

## Decomposing the expresion to the minimum value

As other posts I wrote, we're going to start decomposing an arithmetic expression to the minimum value.
This is the most important step on FPP, where we work composing little functions into higher systems.
Like we saw on the validators post, to build model validators we start decomposing the validator to the most simple
expression and then start to grouping them into a bigger ones.

So, what is the most simple expression inside an arithmetic operation like `2 + 2`? 
The number. The most simple expression is the way to provide a number, and over it, we will build everything.

## The functional way

```swift
() -> Int
```

Now that we have identified what is the most simple expression to start to work on, we can build an expression based on it. Let's write an example.

```swift
typealias Expression = () -> Int

let two: Expression = { 2 }
let operation: Expression =  { two() + two() }
```

As we saw, a function that returns a number can wrap the most basic expression, give us a number, and the `add`
operation. But to be honest, this is far from been usable. So, like other times, let's wrap that function inside a type.
Wrapping functions inside types allow us to add functionality to the type while the base work of our 
functions still keeps simple.

```swift
struct Expression {
  let run: () -> Int
}

let two = Expression { 2 }
let operation = Expression { two.run() + two.run() }
```

Still far from been usable. We are building the `add` operation inside the expression, but we could improve the design naming
inside Expression the add operation. 

```swift
struct Expression {
  let run: () -> Int
}

extension Expression {
  func add(_ expression: Expression) -> Expression {
    .init { self.run() + expression.run() }
  }
}

let two = Expression { 2 }
let operation = two.add(two)
```

And like the add, we are going to add the substract function.

```swift
struct Expression {
  let run: () -> Int
}

extension Expression {
  func add(_ expression: Expression) -> Expression {
    .init { self.run() + expression.run() }
  }
  
  func sub(_ expression: Expression) -> Expression {
    .init { self.run() - expression.run() }
  }
}

let two = Expression { 2 }
let operation = two.add(two).sub(two)
```

And we can extend the Expression type to allow as many operations as we want.
But what if we want to work with other type of values instead only Int. We can make it generic.

```swift
struct Expression<Value> {
  let run: () -> Value
}

extension Expression where Value == Int {
  func add(_ expression: Expression) -> Expression {
    .init { self.run() + expression.run() }
  }
  
  func sub(_ expression: Expression) -> Expression {
    .init { self.run() - expression.run() }
  }
}

let two = Expression { 2 }
let operation = two.add(two).sub(two)
```

At this point, `Expression` is built, and let us extend it with more operations without too much efforrt and we can limit those extensions bassed on the type of the `Expression`.

*What if we want to print the built expression? (like 2+2)*

At this point, we find that our model doesn't allow it. 
Our model has coupled how we build the model and how the model works. This means we're coupling our model with his interpretation. Being the interpretation the way of our model behaves. 

So, how can we fix it?

## The enum choice.

The model proposed by point free on the (chapter)[https://www.pointfree.co/episodes/ep26-domain-specific-languages-part-1] split the model from the interpretation, building only the expression based on the operations, but not applying any logic to it.

It starts with from the same point, what is the most simple expression inside `2 + 2`?. The `Int` value, so let's build a model that allow us to represent it.

```swift
enum Expression {
  case value(Int)
}
```

And then, like we have the value, we can add to our Expression model the operation values. Those operations will be cases of our enum. 

```swift
enum Expression {
  case value(Int)
  case add
  case sub
}
```

Our operations are defined, but them still need parameters to work. The paremeters of the operation could be Int, but our model already allow us to express an Int as an Expression, in our `.value` case. So let's add the parameters needed for the operations.

```swift
indirect enum Expression { // indirect will allow us to have recursive enum cases
  case value(Int)
  case add(Expression, Expression)
  case sub(Expression, Expression)
}
```

Now we can build up our Expression with add and sub, but also our model let us concatenate more than one operation, because an Expression could be a value or a set of operations wrapped inside it.

```swift
let expression: Expression = .add(.value(2), .add(.value(2), .value(2)))
```

Our model is done. Now is time to execute it. So we can have the next function:

```swift
func evaluate(expression: Expression) -> Int {
  switch operation {
    case let .value(value):
        return value
    case let .add(aOP, bOP):
        return evaluate(aOP) + evaluate(bOP)
    case let .sub(aOP, bOP):
        return evaluate(aOP) - evaluate(bOP)
    }
}
```

Okey, but in a real world, we can't have a lot of functions spreaded on all our code, so let's wrap it inside a type. We have to pay speccial attention to `evaluate` function because is a recursive function, so we will need a way to create a type that will be created with a function that receive is own type as parameter. This is needed to be able to call itself recursively.

```swift
struct Runner {
    private let _run: (Runner, Expression) -> Int

    init(_ run: @escaping (Runner, Expression) -> Int) {
        self._run = run
    }

    func run(_ operation: Expression) -> Int {
        _run(self, operation)
    }
}
```

This way we're able to have an evaluator runner inside our `Runner` type with the dot notation available.

```swift
extension Runner {
  static let evaluator = Runner { runner, expression in
    switch operation {
    case let .value(value):
        return value
    case let .add(aOP, bOP):
        return runner.run(aOP) + runner.run(bOP)
    case let .sub(aOP, bOP):
        return runner.run(aOP) - runner.run(bOP)
    case let .mul(rExpr, lExpr):
        return runner.run(rExpr) * runner.run(lExpr)
    }
  }
}
```

Finally, the usage of this is really simple:

```swift
let expression: Expression = .add(.value(2), .add(.value(2), .value(2)))
let result = Runner.evaluator.run(expression)
```

As we have the model interpretation outside wrapped on the `Runner` type, now we can build another runner that will print the expression instead of evaluating it. Like we want to get an `String` now instead a number, we need to make the output of our `Runner` generic.

```swift
struct Runner<Output> {
    private let _run: (Runner, Expression) -> Output

    init(_ run: @escaping (Runner, Expression) -> Output) {
        self._run = run
    }

    func run(_ operation: Expression) -> Output {
        _run(self, operation)
    }
}
```

The generic types doesn't allow us to have `static let` inside if we don't limit the wrapping generic types to concrete ones. So our evaluator runner will looks:

```swift 
extension Runner where Output == Int {
    static let evaluator = Runner<Int> { runner, operation in
        switch operation {
        case let .value(value):
            return Double(value)
        case let .add(aOP, bOP):
            return runner.run(aOP) + runner.run(bOP)
        case let .sub(aOP, bOP):
            return runner.run(aOP) - runner.run(bOP)
        }
    }
}
```

With our types already updated, let's built the printer runner.

```swift
extension Runner where Output == String {
  static let printer = Runner<String> { runner, operation in
    switch operation {
    case let .value(value):
        return value
    case let .add(aOP, bOP):
        return "\(runner.run(aOP)) + \(runner.run(bOP))"
    case let .sub(aOP, bOP):
        return "\(runner.run(aOP)) - \(runner.run(bOP))"
    }
  }
}
```

The usage will be like this.

```swift
let expression: Expression = .add(.value(2), .add(.value(2), .value(2)))
let result = Runner.evaluator.run(expression)
let stringResult = Runner.printer.run(expression)
```

To improve how we run the expressions, we could extend our Expression type a little bit:

```swift
extension Expression {
    func evaluated<Output>(by runner: Runner<Output>) -> Output {
        runner.run(self)
    }
}
```

Now the usage will be.

```swift
let expression: Expression = .add(.value(2), .add(.value(2), .value(2)))
let result = expression.evaluated(by: .evaluator)
let stringResult = expression.evaluated(by: .printer)
```

Another cool improvement we can make is to don't close the value type of our `Expression` model to an `Int` value. So let's make the `.value` case generic and update all our components.

```swift
indirect enum Expression<Value> {
    case value(Value)
    case add(Expression, Expression)
    case sub(Expression, Expression)
}

struct Runner<Input, Output> {
    private let _run: (Runner<Input, Output>, Expression<Input>) -> Output

    init(_ run: @escaping (Runner<Input, Output>, Expression<Input>) -> Output) {
        self._run = run
    }

    func run(_ operation: Expression<Input>) -> Output {
        _run(self, operation)
    }
}

extension Runner where Input == Int, Output == Double {
    static let evaluatorRunner = Runner<Int, Double> { runner, operation in
        switch operation {
        case let .value(value):
            return Double(value)
        case let .add(aOP, bOP):
            return runner.run(aOP) + runner.run(bOP)
        case let .sub(aOP, bOP):
            return runner.run(aOP) - runner.run(bOP)
        }
    }
}

extension Runner where Input == Int, Output == String {
    static let printRunner = Runner<Int, String> { runner, operation in
        switch operation {
        case let .value(value):
            return "\(value)"
        case let .add(aOP, bOP):
            return "\(runner.run(aOP)) + \(runner.run(bOP))"
        case let .sub(aOP, bOP):
            return "\(runner.run(aOP)) - \(runner.run(bOP))"
        }
    }
}
```

Now we can build Expressions not only for Int type but also for any other type that we want to work on. 

## Mapping

Expression has a generic type inside it, this allow us to have the same expressions with different types inside, maybe we even need to transform them. Tranforming an `Expression<Int>` to `Expression<String>` could be really simple if we implement high order functions like map operation inside our `Expression` type.

```swift
extension Expression {
    func map<NewValue>(_ transform: (Value) -> NewValue) -> Expression<NewValue> {
        switch self {
        case .value(let value):
            return .value(transform(value))
        case let .add(lExpression, rExpression):
            return .add(lExpression.map(transform), rExpression.map(transform))
        case let .sub(lExpression, rExpression):
            return .sub(lExpression.map(transform), rExpression.map(transform))
        }
    }
}
```

As you may see, the key point of the map, as on the other functions we've built to use oeprations, is the recurisivity on it due to the indirect cases of the enum. We are interating over all the expressions inside the initial expression and only mutate the value case.

Mutating an expression will be like:

```swift
let expression: Expression<Int> = .add(.value(2), .add(.value(2), .value(2)))
let stringExpression = expression.map { "\($0)" }
```

## Adding Vars

Like the people from Pointfree does, let's add the var case. This case allow us to create vars inside our expression that will be suplied by values when evaluating them.

```swift
indirect enum Expression { // indirect will allow us to have recursive enum cases
  case value(Int)
  case add(Expression, Expression)
  case sub(Expression, Expression)
  case `var`(String)
}
```

The Runner object also needs to be update, it needs to receive a Dictionary of String and values, but unlike pointfree, instead of Ints, we will use Expression. 

Using expressions as the value of the variables, if you saw the video, will avoid us to implement the `bind` case.

The bind case was a special case to bind an Expression to a var, but it add complexity and is really unneded if we model the dictionary to have Expressions. Also, thanks to the Runner generic type, we can limit the dictionary Expression to only allow Expressions with the same type. 

Finally our code look like:

```swift
indirect enum Expression<Value> {
    case value(Value)
    case add(Expression, Expression)
    case sub(Expression, Expression)
    case `var`(String)
}

extension Expression {
    func map<NewValue>(_ transform: (Value) -> NewValue) -> Expression<NewValue> {
        switch self {
        case .value(let value):
            return .value(transform(value))
        case let .add(lExpression, rExpression):
            return .add(lExpression.map(transform), rExpression.map(transform))
        case let .sub(lExpression, rExpression):
            return .sub(lExpression.map(transform), rExpression.map(transform))
        case let .var(key):
            return .var(key)
        }
    }
     
    func evaluate<Output>(_ runner: Runner<Value, Output>, environment: [String: Expression] = [:]) -> Output {
        runner.run(self, environment: environment)
    }
}

struct Runner<Input, Output> {
    private let _run: (Runner<Input, Output>, Expression<Input>, [String: Expression<Input>]) -> Output

    init(_ run: @escaping (Runner<Input, Output>, Expression<Input>, [String: Expression<Input>]) -> Output) {
        self._run = run
    }

    func run(_ operation: Expression<Input>, environment: [String: Expression<Input>] = [:]) -> Output {
        _run(self, operation, environment)
    }
}

extension Runner where Input == Int, Output == Double {
    static let evaluatorRunner = Runner<Int, Double> { runner, operation, environment in
        switch operation {
        case let .value(value):
            return Double(value)
        case let .add(aOP, bOP):
            return runner.run(aOP, environment: environment) + runner.run(bOP, environment: environment)
        case let .sub(aOP, bOP):
            return runner.run(aOP, environment: environment) - runner.run(bOP, environment: environment)
        case let .var(key):
            guard let value = environment[key] else {
                fatalError("Not found Expression for var \(key)")
            }
            return runner.run(value, environment: environment)
        }
    }
}

extension Runner where Input == String, Output == String {
    static let printRunner = Runner<String, String> { runner, operation, environment in
        switch operation {
        case let .value(value):
            return value
        case let .add(aOP, bOP):
            return "\(runner.run(aOP, environment: environment)) + \(runner.run(bOP, environment: environment))"
        case let .sub(aOP, bOP):
            return "\(runner.run(aOP, environment: environment)) - \(runner.run(bOP, environment: environment))"
        case let .var(key):
            guard let value = environment?[key] else {
                fatalError("Not found Expression for var \(key)")
            }
            return runner.run(value, environment: environment)
        }
    }
}
```

Then use the vars will be:

```swift
let expression: Expression<Int> = .add(.var("x"), .value(6))
let result = expression.evaluate(.evaluatorRunner, environment: ["x": .value(2)])
```

## Friendly sintax

The model is working. The interpretation is abstracted from the model creation. That's okey, but, why if we're building an expression based on arithmetic operations, don't we use operators?

In order to do this, we're going to extend the `Expression` model to be `ExpressibleByIntegerLiteral` limited to the Int expression. 

```swift
extension Expression: ExpressibleByIntegerLiteral where Value == Int {
    init(integerLiteral value: IntegerLiteralType) {
        self = .value(value)
    }
}
```

To build the `.var` case, we're going to implement the `ExpressibleByStringLitleral`, this has a trade-off that we must consider. We don't be able to build an `Expression<String>` from litlerals, because we will use the string literal to build the var case on any type of Expression.

```swift
extension Expression: ExpressibleByStringLiteral {
    init(stringLiteral value: StringLiteralType) {
        self = .var(value)
    }
}
```

And then, we only need to create the operators.

```swift
extension Expression {
    static func +(lhs: Expression, rhs: Expression) -> Expression {
        .add(lhs, rhs)
    }
    
    static func -(lhs: Expression, rhs: Expression) -> Expression {
        .sub(lhs, rhs)
    }
}
```

Now, we can build Expressions like if we are writting code.

```swift
let expression: Expression<Int> = 6 + 2 + "x"
let result = expression.evaluate(.evaluatorRunner, environment: ["x": .value(2)])
```

## Conclusion

At the end, all of these ways of coding looks pretty similar. First you have to find what is the minimum expression to your goal, usually that minimum expression will be a function. Then wrap it inside a type, and then extend the type to compose or whatever.

For the functional choice, we've started from `() -> Int`, wrapped it inside a Type. Added a couple of functions inside the type to allow `add` and `sub`, and made it generic. I didn't implements the oeprators and `Expressible` protocols inside this case, but it could be implemented too like in the `enum` choice.

On the other hand, for the enum choice, we've started from the most simple case `.value(Int)` and then add the other cases. We've created the runner to wrap inside an object how the Expression is interpreted, based on the function `(Expression) -> Int`. Then we've updated it to be generic. Added the `.var` case with the environment dictionary, and some sugar syntax to write the expressions with operators.

<br/>
<br/>

### All the code

```swift
import Foundation

indirect enum Expression<Value> {
    case value(Value)
    case add(Expression, Expression)
    case sub(Expression, Expression)
    case mul(Expression, Expression)
    case `var`(String)
}

extension Expression {
    func map<NewValue>(_ transform: (Value) -> NewValue) -> Expression<NewValue> {
        switch self {
        case .value(let value):
            return .value(transform(value))
        case let .add(lExpression, rExpression):
            return .add(lExpression.map(transform), rExpression.map(transform))
        case let .sub(lExpression, rExpression):
            return .sub(lExpression.map(transform), rExpression.map(transform))
        case let .mul(lExpression, rExpression):
            return .mul(lExpression.map(transform), rExpression.map(transform))
        case let .var(key):
            return .var(key)
        }
    }
     
    func evaluate<Output>(_ runner: Runner<Value, Output>, environment: [String: Expression]? = nil) -> Output {
        runner.run(self, environment: environment)
    }
}

extension Expression: ExpressibleByIntegerLiteral where Value == Int {
    init(integerLiteral value: IntegerLiteralType) {
        self = .value(value)
    }
}

extension Expression: ExpressibleByStringLiteral {
    init(stringLiteral value: StringLiteralType) {
        self = .var(value)
    }
}

extension Expression {
    static func +(lhs: Expression, rhs: Expression) -> Expression {
        .add(lhs, rhs)
    }
    static func -(lhs: Expression, rhs: Expression) -> Expression {
        .sub(lhs, rhs)
    }
    static func *(lhs: Expression, rhs: Expression) -> Expression {
        .mul(lhs, rhs)
    }
}

struct Runner<Input, Output> {
    private let _run: (Runner<Input, Output>, Expression<Input>, [String: Expression<Input>]?) -> Output

    init(_ run: @escaping (Runner<Input, Output>, Expression<Input>, [String: Expression<Input>]?) -> Output) {
        self._run = run
    }

    func run(_ operation: Expression<Input>, environment: [String: Expression<Input>]? = nil) -> Output {
        _run(self, operation, environment)
    }
}

extension Runner where Input == Int, Output == Double {
    static let evaluatorRunner = Runner<Int, Double> { runner, operation, environment in
        switch operation {
        case let .value(value):
            return Double(value)
        case let .add(aOP, bOP):
            return runner.run(aOP, environment: environment) + runner.run(bOP, environment: environment)
        case let .sub(aOP, bOP):
            return runner.run(aOP, environment: environment) - runner.run(bOP, environment: environment)
        case let .mul(rExpr, lExpr):
            return runner.run(rExpr, environment: environment) * runner.run(lExpr, environment: environment)
        case let .var(key):
            guard let value = environment?[key] else {
                fatalError("Not found Expression for var \(key)")
            }
            return runner.run(value, environment: environment)
        }
    }
}

extension Runner where Input == String, Output == String {
    static let printRunner = Runner<String, String> { runner, operation, environment in
        switch operation {
        case let .value(value):
            return value
        case let .add(aOP, bOP):
            return "\(runner.run(aOP, environment: environment)) + \(runner.run(bOP, environment: environment))"
        case let .sub(aOP, bOP):
            return "\(runner.run(aOP, environment: environment)) - \(runner.run(bOP, environment: environment))"
        case let .mul(rExpr, lExpr):
            return "\(runner.run(rExpr, environment: environment)) * \(runner.run(lExpr, environment: environment))"
        case let .var(key):
            guard let value = environment?[key] else {
                fatalError("Not found Expression for var \(key)")
            }
            return runner.run(value, environment: environment)
        }
    }
}

let expression: Expression<Int> =  6 + 2 + "x"
let result = expression.evaluate(.evaluatorRunner, environment: ["x": .value(2)])
```