---
layout: post
title: Highly Composable Validators
author: juli
---

How many times, we need to validate a form, get usable errors for that validation rules, validate some user data while the user inputs it...

This time we're going to cover how to build little but composable Validators. We're going to start decomposing a Validator to is most simple code unit, and then we'll start to build over that until we build our models validators, and of course, making them easy to test.

So let's start thinking about what it's a validator.

The most simple code unit to represent a validator it's a function that given a value will tell us if the value it's valid or not. We can represent this like: `(Value) -> Bool`. This could be our start point. 

To make this more understandable, let's put an example where we have to validate an `User`. We will need to validate his `email`, and his phone number (based on spanish rules, 9 digits).

```swift
struct User {
    let email: String
    let phone: String 
}
```

So, with the previous validator unit code, we will have:

```swift

let validateEmail: (String) -> Bool = { email in
    email.contains("@")
}

let validatePhone: (String) -> Bool = phone {
    let numberSet = CharacterSet(charactersIn: "0123456789")
    let phoneSet = CharacterSet(charactersIn: phone)
    return phone.count == 9 && phoneSet.isSubset(of: numberSet)
}

```

These two functions will return us a boolean if the String is a valid email or phone, obviously this validations are dumb examples.

But now, would be better if we have that funtions wrapped inside an object that validates. So let's create an object that will validate a value.

```swift
struct Validator<Value> {
    private let validation: (Value) -> Bool

    func validate(_ value: Value) -> Bool {
        validation(value)
    }
}
```

Now that we have a very simple struct to build our Validators, we can start extend it to name those validators and wrap them over Validators of the Value they're validating. 

Then our previous validations functions will be:

```swift
extension Validator where Value == String {
    static let emailValidator = Validator<String> { email in
        email.contains("@")
    }

    static let phoneValidator = Validator<String> { phone in
        let numberSet = CharacterSet(charactersIn: "0123456789")
        let phoneSet = CharacterSet(charactersIn: phone)
        return phone.count == 9 && phoneSet.isSubset(of: numberSet)
    }
}
```

With this simple struct to wrap our validations, now we can use the dot notation to refer our validators, wherever we need use them.

```swift
let emailValidator: Validator<String> = .emailValidator
```

## Composing validators

Our validator it's a type. It's a type that wraps a function, and thanks to that we can start composing them in a simple way. In order to do this we only need to extend our Validator type and create a function to compose them.

```swift
extension Validator {
    static func compose(first: Validator<Value>, second: Validator<Value>) -> Validator<Value> {
        Validator<Value> { value in
            first(value) && second(value)
        }
    }
}
```

With that simple extension we're creating a new validator that wraps the previous two validators. This way we can compose more complex Validator based on the simples ones. We keep the testability easy because we can test each simple rule, the compose method, and we avoid to test the composed ones, that will be harder to test.

But we keep with one problem, we only can compose from two to other two, and we're looking for compose as many as we want. So let's build a new function to compose a bunch of them.

```swift 
static func compose(_ validators: Validator<Value>...) -> Validator<Value> {
    guard let first = validators.first else { preconditionFailure("Empty validators") }
    return validators.dropFirst().reduce(first) { compose(first: $0, second: $1) }
}
```

With that simply function we can compose into one Validator as many validators as we want. The dot notation is still available.

To show how we can use it, let's create another validator to assert that one string is not empty, and then we will add that validator to the email validator.

```swift 
extension Validator where Value == String {
    static let containsAt = Validator<String> { string in
        string.contains("@")
    }
    
    static let notEmptyString = Validator<String> { string in
        !string.isEmpty
    }

    // the composed one
    static let emailValidator = Validator<String>.compose(
        .containsAt,
        .notEmptyString
    )
}
```

As you may see, we're creating very simples validators with one rule and then composing as many rules as we want to create more complex validators, but at this moment, we have one problem, we can't know if the validation fails what is the failing rule.

Very inspired on the [point free validated](https://github.com/pointfreeco/swift-validated) I took the `Validated` obect to handle the validated value or the validation error if it fails, but with a little difference, I like that if a validation fails, only return the first error, not all. Because I (this is very personal opinion) don't like that forms that you start to input one value and all the form starts failing.

So, let's create the Validated object and the adapt our validator to use it.

```swift
enum Validated<Value, Error> {
    case valid(Value)
    case invalid(Error)
}
```

As you may see, this error is not implementing Swift.Error, because we don't want to throw anything.

Now that we have the `Validated` object, let's update our validator to work with it. The main key of this is to return the first error that we get validating and like we will see, now our validators, validate a type and could give us a ValidationError typed.


```swift
struct Validator<Value, Error> {
    let validation: (Value) -> Validated<Value, Error>

    func validate(_ value: Value) -> Validated<Value, Error> {
        validation(value)
    }
}
```

At this moment, we have to update how our `compose` functions works.

```swift
//The first compose function to wrap two validators into a new one.
private func compose(first: Validator<Value, Error>, second: Validator<Value, Error>) -> Validator<Value, Error> {
    Validator<Value, Error> {
        switch first($0) {
        case .valid(let value): return second(value)
        case .invalid(let error): return .invalid(error)
        }
    }
}

//The second compose function to wrap as many validators as we need into a new one.
static func compose(_ validators: Validator<Value, Error>...) -> Validator<Value, Error> {
    guard let first = validators.first else { preconditionFailure("Empty validators") }
    return validators.dropFirst().reduce(first) { compose(first: $0, second: $1) }
}
```

I've set as private the first `compose` function to clean how the valdiators are composed and provide only one function to compose them. 

Now let's update how the phone and email validator works.

```swift
enum StringValidationError {
    case emptyString
    case notValidEmail
    case phoneInvalidLength
    case mustContainsOnlyNumbers
}

extension Validator where Value == String, Error == StringValidationError {
    static let notEmpty = Validator<String, StringValidationError> { string in
        string.isEmpty ? .invalid(.emptyString) : .valid(string)
    }

    static let containsAt = Validator<String, StringValidationError> { string in
        string.contains("@") ? .valid(string) : .invalid(.notValidEmail)
    }

    static let phoneLength = Validator<String, StringValidationError> { string in
        string.count == 9 ? .valid(string) : .invalid(.phoneInvalidLength)
    }

    static let onlyNumbers = Validator<String, StringValidationError> { string in
        let numberSet = CharacterSet(charactersIn: "0123456789")
        let stringSet = CharacterSet(charactersIn: string)
        return stringSet.isSubset(of: numberSet) ? .valid(string) : .invalid(.mustContainsOnlyNumbers)
    }

    static let phoneValidator = Validator<String, StringValidationError>.compose(
        .notEmpty,
        .phoneLength,
        .onlyNumbers
    )

    static let emailValidator = Validator<String, StringValidationError>.compose(
        .containsAt,
        .notEmpty
    )
}
```

At this point, we know if the validation fail, what is the produced error.

Okey, but now, how we can compose from this primitive types validators to a bigger one?

## Compose them to validate complex models

Back to the user model proposed previously. We need a way to compose a validator of the User model, that validates each path (if needed) of the user, and not only that, when we're validating the user, we want to get errors of the `UserValidationError` domain, not from the `StringValidationError` so we will also need a new way transform the path domain `ValidationError` to the Model domain `ValidationError`.

To create the user `Validator` from other validators we need a function to compose a Model `Validator` that takes the value from a model's path, and validates it with the provided validator and then if error it's produced, mutate it from the path domain error to the model domain error.

I know this sounds a little bit hard, but the function it's reallly simple. What we need? The path, the validator and the mutating func, so let's create the function that given those values prodive us the new `Validator`. This function will be also on the `Validator` object to keep the dot notation and be able to use it when composing.

```swift
extension Validator {
    static func path<SubValue, SubError>(_ keypath: KeyPath<Value, SubValue>,
                                         _ validation: Validator<SubValue, SubError>,
                                         mapError: @escaping (SubError) -> Error) -> Validator<Value, Error> {
        Validator<Value, Error> { value in
            switch validation(value[keyPath: keypath]) {
            case .valid: return .valid(value)
            case .invalid(let error): return .invalid(mapError(error))
            }
        }
    }
}
```

This is the key point of our `Validator`, with that simple function, we can compose simple validators into a biggers validators to validate models from the path values.

Now we can create a user validator that uses the email and phone validator to validate the required paths.

```swift
enum UserValidationError {
    case .invalidEmail(StringValidationError)
    case .invalidPhone(StringValidationError)
}

extension Validator where Value == User, Error = UserValidationError {
    static let userValidator = Validator<User, UserValidationError>.cmpose(
        path(\.email, .emailValidator, mapError: { .invalidEmail($0) }),
        path(\.phone, .phoneValidator, mapError: { .invalidPhone($0) })
    )
}
```

And now, we can see what path it's failing, and what it's the failure of the path. As the map error it's a simple function that given one error returns a new error type, the possibilities of that mapping are endless. For example, if we only want to show to the user a message, our `StringValidationError` could have a `failulreReason: String` with a message like "Can't be empty", and we can compose the messages to do something like `mapError: { .invalidEmail("email \($0.failureReason") }, or whatever.

## Conclusions

We've seen how creating simple Validation rules, we can compose them to create more complex validators like `email` or `phone` based on two or more rules, and then, go even further and compose those Validators to create a Model type validator that validates each path of the model with painless, keeping simplicity and readability.

These type of validaros keep being easy testable, because they're based on simple rules and we can avoid test the composed ones, because if the compose function is tested and the simples rules are tested too, the composed one will work fine always. Not forget that if you're mapping errors with a lot of logic, you should test that error mappings.