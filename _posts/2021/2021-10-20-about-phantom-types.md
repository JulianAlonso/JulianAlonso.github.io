---
layout: post
title: About Phantom Types
author: juli
---

Let's say we have to call an external SDK in order to perform any kind of work. This SDK requires a token, and this token is a String.

So we have a function to call the SDK called:

`sdk.work(accessToken: String)`

And our design wraps that function into:

`work(accessToken: String)`

The way to get that token is to call `GET: /api/sdk/accessToken`, with the following response:

```swift
{
    "access_token": "0000"
}
/// mapped to:
struct SDKGetTokenResponse: Decodable {
    let accessToken: String
}
```

And that's fine, then X service returns the `String` and that's the thing we work with. The String Type.
While we're building this we have enough context to know the String type is returned from the SDKGetToken service and we use it in the `work` use case. 

Maybe this is fine, and compiles and works but the next time we read the work function, two months later, we think, WTF is that string?, where does it come from? We only can follow the flow with the caller's function, if Xcode wants work that day, but we can't perform any search over the String type, because we will have tons of strings on our code.

## Phantom Types.

A phantom type is a generic type in which implementation we don't use the generic type:
- Encode information about how and where a value can be used
- Help us to provide more information on function signatures

Given the previous example, what if instead of having that `accessToken` as a String we wrap it inside `AccessToken` ?


```swift
{
    "access_token": "0000"
}
/// mapped to:
enum SDK {}

struct GetTokenResponse<SDKName>: Decodable {
    let accessToken: AccessToken<SDKName>
}

struct AccessToken<SDKName> {
    let rawValue: String
}

extension AccessToken: Decodable {
    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        self.init(rawValue: try container.decode())
    }
}
```

The function `work` will look like:

```swift
func work(accessToken: AccessToken<SDK>) {
    SDK.work(accessToken.rawValue)
}
```

Creating the AccessToken structure with a Phantom Type, our code is more concise, reading the type we know what it is, and we can perform a global search to know where it comes from, where it is used, etc.

Even if we're using more than one SDK, all that we need to do to extend our code is to create another SDKName enum and our work is done:

```swift
{
    "access_token": "0000"
}
/// mapped to:
enum SDK {}
enum AnotherSDK {}

struct GetTokenResponse<SDKName>: Decodable {
    let accessToken: AccessToken<SDKName>

    enum CodingKeys: String, CodingKey {
        case accessToken = "access_token"
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.accessToken = .init(rawValue: try container.decode(String.self, forKey: .accessToken))
    }
}

struct AccessToken<SDKName> {
    let rawValue: String
}

func work(accessToken: AccessToken<SDK>) {
    SDK.work(accessToken.rawValue)
}


func anotherWork(accessToken: AccessToken<AnotherSDK>) {
    AnotherSDK.work(accessToken.rawValue)
}

```

For testing, we can extend that type with a `static let stub` to perform asserts with the dot notation, and only one option will be available, instead of having a lot of values inside String type. 

But our code will be so conscious that we won't be able to compare two AccessToken if our SDKName generic type doesn't match.
