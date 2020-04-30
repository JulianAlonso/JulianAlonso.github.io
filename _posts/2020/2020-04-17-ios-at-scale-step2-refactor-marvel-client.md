---
layout: post
title: Xcode at scale - Refactor Marvel - Client applyin IoC
author: juli
---

This is the third step of a serie named **iOS at Scale** based on the next steps:
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


On this step we will review MarvelClient Module and apply to it some new techniques to make it more readable, also we will change how its structured, and talk about some key points inside an `HTTPClient`. Also, we're going to end up with one Library out, `Networking`, we will see this proccess it's so easy thanks to [SPM](). This library will allow us to share the same HTTPClient along the whole company, in a painless way.

Take the project and go to step-1 
(As you can see, a Promise framework it's done, but we are not going to use them until the last series step).

## Refactor the actual code.

Open the MarvelAPIClient and then...
```swift
public class MarvelAPIClient {
    private let baseEndpointUrl = URL(string: "https://gateway.marvel.com:443/v1/public/")!
    private let session = URLSession(configuration: .default)
    
    private let publicKey: String
    private let privateKey: String
    
    public init(publicKey: String, privateKey: String) {
        self.publicKey = publicKey
        self.privateKey = privateKey
    }
    
    /// Sends a request to Marvel servers, calling the completion method when finished
    public func send<T: APIRequest>(_ request: T, completion: @escaping ResultCallback<DataContainer<T.Response>>) {
        let endpoint = self.endpoint(for: request)
        
        let task = session.dataTask(with: URLRequest(url: endpoint)) { data, response, error in
            if let data = data {
                do {
                    // Decode the top level response, and look up the decoded response to see
                    // if it's a success or a failure
                    let marvelResponse = try JSONDecoder().decode(MarvelResponse<T.Response>.self, from: data)
                    
                    if let dataContainer = marvelResponse.data {
                        completion(.success(dataContainer))
                    } else if let message = marvelResponse.message {
                        var code = 0
                        if let httpResponse = response as? HTTPURLResponse {
                            code = httpResponse.statusCode
                        }
                        completion(.failure(MarvelError.server(code: code,
                                                               message: message)))
                    } else {
                        completion(.failure(MarvelError.decoding))
                    }
                } catch {
                    completion(.failure(error))
                }
            } else if let error = error {
                completion(.failure(error))
            }
        }
        task.resume()
    }
    
    /// Encodes a URL based on the given request
    /// Everything needed for a public request to Marvel servers is encoded directly in this URL
    private func endpoint<T: APIRequest>(for request: T) -> URL {
        guard let baseUrl = URL(string: request.resourceName, relativeTo: baseEndpointUrl) else {
            fatalError("Bad resourceName: \(request.resourceName)")
        }
        
        var components = URLComponents(url: baseUrl, resolvingAgainstBaseURL: true)!
        
        // Common query items needed for all Marvel requests
        let timestamp = "\(Date().timeIntervalSince1970)"
        let hash = "\(timestamp)\(privateKey)\(publicKey)".md5
        let commonQueryItems = [
            URLQueryItem(name: "ts", value: timestamp),
            URLQueryItem(name: "hash", value: hash),
            URLQueryItem(name: "apikey", value: publicKey)
        ]
        
        // Custom query items needed for this specific request
        let customQueryItems: [URLQueryItem]
        
        do {
            customQueryItems = try URLQueryItemEncoder.encode(request)
        } catch {
            fatalError("Wrong parameters: \(error)")
        }
        
        components.queryItems = commonQueryItems + customQueryItems
        
        // Construct the final URL with all the previous data
        return components.url!
    }
}

```

This is a HUGE Client.

The first to note it we have two responsabilities inside the HTTPClient. Create the endpoint given a request and making the request. So, the first thing we're going to do its extracting the endpoint creation out, this way we reduce a bit the code in this file.

As we can se, endpoint it's a function that given a `Request`, creates and returns an URL. So, maybe, this could be inside an extension of `APIRequest` itself, and take out the parameter of the function because we will use self.

We finally got this: (commit: `e80bdbf`)

```swift
extension APIRequest {
    func endpoint(base url: URL, publicKey: String, privateKey: String) -> URL {
        guard let baseUrl = URL(string: resourceName, relativeTo: url) else {
            fatalError("Bad resourceName: \(resourceName)")
        }
        
        var components = URLComponents(url: baseUrl, resolvingAgainstBaseURL: true)!
        
        let timestamp = "\(Date().timeIntervalSince1970)"
        let hash = "\(timestamp)\(privateKey)\(publicKey)".md5
        let commonQueryItems = [
            URLQueryItem(name: "ts", value: timestamp),
            URLQueryItem(name: "hash", value: hash),
            URLQueryItem(name: "apikey", value: publicKey)
        ]
        

        let customQueryItems: [URLQueryItem]
        do {
            customQueryItems = try URLQueryItemEncoder.encode(self)
        } catch {
            fatalError("Wrong parameters: \(error)")
        }
        
        components.queryItems = commonQueryItems + customQueryItems
        
        return components.url!
    }
}
```

Now back on the Client, the code to decode the response, it's hard to follow, has five calls to `completion` and a lot of nested `if`s.
First, let's think what are the options to unwrap a response.

There are three options:
- data without http code error
- data with http code error
- only the error.

So lets build a switch to handle this options. (commit: `ff59ecc`)

```swift
switch (data, response, error) {
case let (.some(data), .some(response), _) where response.hasOkStatus: break
case let (.some(data), .some(response), _) where !response.hasOkStatus: break
default: break
}
```

```swift
extension URLResponse {
    var okStatuses: [Int] { Array(200...299) }
    var hasOkStatus: Bool {
        guard let response = self as? HTTPURLResponse else { return false }
        return okStatuses.contains(response.statusCode)
    }
}
```

Now, we made easier the response parsing, and we know on every moment what we want to parse.
The MarvelResponse it's a little bit _tricky_, beacuse it groups the error and the data, and both are nullables.

One of the things that we must try to avoid are the usage of `Optionals` properties when they're not optionals. This will lead us to have a lot of `if let` on our code that we don't need and make checks that really are impossible states. If we got the data, we aren't going to have `Error` and vice versa.

```swift
/// Top level response for every request to the Marvel API
/// Everything in the API seems to be optional, so we cannot rely on having values here
public struct MarvelResponse<Response: Decodable>: Decodable {
    /// Whether it was ok or not
    public let status: String?
    /// Message that usually gives more information about some error
    public let message: String?
    /// Requested data
    public let data: DataContainer<Response>?
}
```

Lets split the `Response` from the `Error`. And one more thing, let's use namespaces, so `MarvelResponse` will be `Response` and the new `Error` will be `Error`.  Using namespaces we avoid to write long names that are hard to remember. Since swift intruduced namespaces this could help us to give better names to code pieces and be more precise.

Also I've removed old code from the `send` method. (commit: `0d29b98`).

```swift
public struct Response<Body: Decodable>: Decodable {
    public let body: DataContainer<Body>?
}
```

```swift
enum Error<T: Decodable>: Swift.Error {
    case known(code: Int, body: T)
    case unkown(code: Int, Error)
    case underlying(Error)
}
```

Now, it's time to parse the Response. I've made `MarvelError` decodable to create the option `.server(code, reason)` decoding.
You can see the app working on commit `ef40cff`. Also I've changed the `DataContainer` to a `Page`, due it is giving us only data about the pagination of the API. 

```swift
public func send<T: APIRequest>(_ request: T, completion: @escaping ResultCallback<Page<T.Response>>) {
    let endpoint = request.endpoint(base: baseEndpointUrl, publicKey: publicKey, privateKey: privateKey)
    
    let task = session.dataTask(with: URLRequest(url: endpoint)) { data, response, error in
        do {
            switch (data, response, error) {
            case let (.some(data), .some(response), _) where response.hasOkStatus:
                let body = try JSONDecoder().decode(Response<T.Response>.self, from: data).body
                completion(.success(body))
                return
            case let (.some(data), .some(response), _) where !response.hasOkStatus:
                let code = response.httpCode!
                let error = try JSONDecoder().decode(MarvelError.self, from: data)
                completion(.failure(Error.known(code: code, body: error)))
                return
            case let (_, _, .some(error)):
                completion(.failure(Error<MarvelError>.underlying(error)))
                return
            default: fatalError("Impossible")
            }
        } catch {
            print(error)
            completion(.failure(Error<MarvelError>.underlying(error)))
        }
    }
    task.resume()
}
```

At this moment, our code has been reduced, we got request's, the client only sends the response, our parsing it's only catching possible states...

But we can even go further.

## Split the client from the Marvel API

One of the best things we can do, due to we are a huge company with a big team, it's to take out our client from the `MarvelAPI`, this way we will share the same _how to make a request_ along all our company and much more profits.

In order to do this, we will create a [SPM Package](https://swift.org/package-manager/) with a few objects. Doing this we will notice that there are some parts of the `MarvelAPI` code very coupled to the `Client`, that other ways we don't discover. So let's start to spliting the client into a new `Library`.

To do this, we create `Libraries/Networking`. All the folders inside `Libraries` will be components to be shared. Then inside that folder, we run `swift package init --type library`, this will create the skeleton of our library. This way we have a place for our app domain modules (`Modules` folder and generic libraries on `Libraries`)(commit: `ad71784`)

Like this library will be only to iOS, and with one target, I'm going to do a little cleanup inside the folder and the `Package.swift`. (commit: `9e3e024`)

Then, add the dependency to our `ProjectDefinitionHelper` on Tuist folder, and then to the `MarvelAPI` module. Notice that to be working on a package under development, the only thing we need to do its to have a reference to that package on our working XCWorkspace, so on `Workspace.swift` we add the folder reference to it. (commit: `d82944b`)

Note: _with Tuist you can also add the package with a path, we will see this on one of the lasts posts_

Time to create our client. I've added an `HTTPClient` and moved some files, (commit: `4793cbd`)

![NetworkingState]({{ basepath }}/resources/images/posts/ios_at_scale/networking-after-move-files.png){: .img-rounded .img-responsive }

Now, our API Request model it's higly coupled to `MarvelAPI` needs, also this `APIRequest` force us to create a child object for each request we need to do. I like the classes approach, we see only looking on one folder how many request we have, but working on a big project we could end up with a lot of classes and code to maintain, so instead of this, we're going to use a function to build the request.

So like our client has a `baseURL` where we concatenate `Endpoints`, let's create an `Endpoint` struct to build requests based on `paths`.

Note: _this is a very basic endpoint, and only to perform get requests and url encoded paramters, but you can extend it as many as you need, I use this approach on production apps_
(commit: `82b126e`)


```swift 
public struct Endpoint {
    let endpoint: String
    var paramters: [String: Any]
}
```

That's it. Now, we need our endpoint to be encodable to a host. (commit: `8e4607a`)

```swift
extension Endpoint {
    func encode(to host: URL) -> URL {
        let url = host.appendingPathComponent(endpoint)
        var components = URLComponents(url: url, resolvingAgainstBaseURL: false)!
        
        components.queryItems = paramters.map { URLQueryItem(name: $0.key, value: "\($0.value)") }
        
        return components.url!
    }
}
```

Now that we have our endpoint, lest change how the HTTPClient works with it. First of all, is make this Generic and typing error:

```swift
public typealias ResultCallback<Value> = (Result<Value, Swift.Error>) -> Void
```
to: 

```swift
public typealias Callback<Value, E: Swift.Error> = (Result<Value, Error<E>>) -> Void
```

This callback will return our `Networking` error, but let us specify the body of the error in case that it exists. 
Then, the client implementation looks like: (commit: `e58f8fc`)

```swift
public func perform<T: Decodable, E: Decodable>(_ endpoint: Endpoint, completion: @escaping Callback<T, Error<E>>) {
    let url = endpoint.encode(to: host)
    
    let task = session.dataTask(with: url) { data, response, error in
        do {
            switch (data, response, error) {
            case let (.some(data), .some(response), _) where response.hasOkStatus:
                let body = try JSONDecoder().decode(T.self, from: data)
                completion(.success(body))
                return
            case let (.some(data), .some(response), _) where !response.hasOkStatus:
                let code = response.httpCode!
                let error = try JSONDecoder().decode(E.self, from: data)
                completion(.failure(.known(code: code, body: error)))
                return
            case let (_, _, .some(error)):
                completion(.failure(.underlying(error)))
                return
            default: fatalError("Impossible")
            }
        } catch {
            completion(.failure(.underlying(error)))
        }
    }
    task.resume()
}
```

Now, we can remove `HTTPParameter.swift`, `URLQueryEncoder.swift` and `APIRequest.swift`. So our client, get's reduced to: (commit: `d0d55af`)

![NetworkingState]({{ basepath }}/resources/images/posts/ios_at_scale/networking-refactor-step1.png){: .img-rounded .img-responsive }

But we have one more thing to do. This client its actually not authenticating our requests. In order to this, we're going to add an pAuthenticationp struct, and to be more generic, a protocol to authenticate an endpoint. This protocol will follow [ guideline](https://swift.org/documentation/api-design-guidelines/) to name protocols and be named `Authorizating`.

```swift
public protocol Authorizating {
    func authenticate(endpoint: Endpoint) -> Endpoint
}
```

That's the protocol. To take an Endpoint and create a new one with more parameters, we're going to add a method to the `Endpoint` like `adding(parameters: [String: Any]) -> Endpoint`. Then add the Authorizating to the creation of the client. And we end up with this client: (commit: `33d08a5`)

```swift
public class HTTPClient {
    
    private let host: URL
    private let session: URLSession
    private let authorization: Authorizating
    
    public init(host: URL, session: URLSession, authorization: Authorizating) {
        self.host = host
        self.session = session
        self.authorization = authorization
    }
    
    public func perform<T: Decodable, E: Decodable>(_ endpoint: Endpoint, completion: @escaping Callback<T, Error<E>>) {
        let url = authorization.authorize(endpoint: endpoint).encode(to: host)
        
        let task = session.dataTask(with: url) { data, response, error in
            do {
                switch (data, response, error) {
                case let (.some(data), .some(response), _) where response.hasOkStatus:
                    let body = try JSONDecoder().decode(T.self, from: data)
                    completion(.success(body))
                    return
                case let (.some(data), .some(response), _) where !response.hasOkStatus:
                    let code = response.httpCode!
                    let error = try JSONDecoder().decode(E.self, from: data)
                    completion(.failure(.known(code: code, body: error)))
                    return
                case let (_, _, .some(error)):
                    completion(.failure(.underlying(error)))
                    return
                default: fatalError("Impossible")
                }
            } catch {
                completion(.failure(.underlying(error)))
            }
        }
        task.resume()
    }
}
```

And the last thing. Lets provide a protocol also with the Client. (commit: `618ace2`)

```swift
public protocol HTTPPerforming {
    func perform<T: Decodable, E: Decodable>(_ endpoint: Endpoint, completion: @escaping Callback<T, Error<E>>)
}
```

So, recapitulation to the moment:
- We've refactorized a little bit the `MarvelAPI` to be cleaner and concise.
- Extracted the HTTPClient to be out of `MarvelAPI`, this way, `MarvelAPI` only have code related to Marvel Server.
- Removed complexity inside `HTTPClient`.
- Added the option to parse the body error in a generic way.
- Added dependency injection to the HTTPClient.
- Added Authorizating to hanlde different ways of Authorization if needed.
- Ability to share the client if we want.

Now, we must adapt our `MarvelAPI` to this client.

As we can see on the code. `DataProvidersKit`, was accessing to the client, passing it the request, and then mutating the data. But we don't need to expose requests and so on. We will expose only a `CharacterProvider` that will perform the requests. Like the `Provider` now are inside `MarvelClient`, we need to rename the `DataProvidersKit` to `Repositories`.

Also, to not couple our `Provider` to the network callback, we need to create inside `Support` a new completion block type. 

```swift
public typealias Done<T, E: Error> = (Result<T, E>) -> Void
```

And now our protocol of the `CharacterProvider` will be `CharacterProviding`.

```swift
public protocol CharacterProviding {
    func characters(_ done: Done<[ComicCharacter], MarvelError>)
    func character(by id: Int, _ done: Done<ComicCharacter, MarvelError>)
}
```

Now the Provider: (commit: `97d6f6e`)
```swift
public final class CharacterProvider: CharacterProviding {
    
    private let client: HTTPPerforming
    
    init(client: HTTPPerforming) {
        self.client = client
    }
    
    public func characters(_ done: (Result<[ComicCharacter], MarvelError>) -> Void) {
        client.perform(Endpoint(path: "/v1/public/characters")) { (result: Result<Page<[ComicCharacter]>, Error<MarvelError>>) in
            
        }
    }
    
    public func character(by id: Int, _ done: (Result<ComicCharacter, MarvelError>) -> Void) {
        client.perform(Endpoint(path: "/v1/public/characters/\(id)")) { (result: Result<Page<[ComicCharacter]>, Error<MarvelError>>) in
            
        }
    }
    
}
```

We are seeing `ComicCharacter.swift`, and since we don't have various Characters type, let's refactor the `ComicCharacter` name to `Character` (commit: `3263e54`), and then, remove the old Request objects that we dont need anymore. (commit: `93f94f9`).

Note: _like we're on swift and we have typenames if two objects have the same name in different modules, we can use `Module.ObjectName` to refer the needed one_

The last, we must provide a way to `Authorize` the client based on the API we're consuming, this `Authorization` it's from our `Marvel` domain, so we must provide it on the current `Module` (`DataProvidersKit`). (commit: `45d8eba`)

```swift
public struct Authorization {
    public let publicKey: String
    public let privateKey: String
    
    public init(publicKey: String, privateKey: String) {
        self.publicKey = publicKey
        self.privateKey = privateKey
    }
}
```

And the `Authorizating` part: (commit: `947d6b3`)

```swift
private extension Authorization {
    var parameters: [String: String] {
        let timeStamp = "\(Date().timeIntervalSince1970)"
        return [
            "apikey": publicKey,
            "hash": "\(timeStamp)\(privateKey)\(publicKey)".md5,
            "ts": timeStamp
        ]
    }
}

extension Authorization: Authorizating {
    public func authorize(endpoint: Endpoint) -> Endpoint {
        endpoint.adding(parameters: parameters)
    }
}
```

Note: _This way of `Authorize` an `Endpoint` it's flexible enough to authorize the endpoint in other ways, adding headers for example or whatever we need._

Now, let's end our `CharacterProvider`. To make our mapping from Error<MaverError>, to `MarvelError` easier inside a `Result`, we need a Result extension that allow us to map the `Error` to a new type of `Error`, and we're going to do that on `Support`, (this way we have available those pieces of code on all our app, and we can take our support module from one project to other). (commit: `8285f6a`)
As we need to different the method name from the `mapError` that will return an `Error`, I've name it `mapTerror` that also it's fun.

```swift
public extension Result {
    func mapTerror<NewFailure>(_ transform: (Failure) -> NewFailure) -> Result<Success, NewFailure> {
        switch self {
        case .success(let value): return .success(value)
        case .failure(let error): return .failure(transform(error))
        }
    }
}
```

And now we're going to extend the `Error<MarvelError>` to add it a funtion that map it to a `MarvelError` directly.

MaverError, needs a new case to wrap also an underlying `Error` (commit: `400f047`)

```swift
extension Error where T == MarvelError {
    var marvelError: MarvelError {
        switch self {
        case .known(_, let body): return body
        case .unkown(_, let error): return .underlying(error)
        case .underlying(let error): return .underlying(error)
        }
    }
}
```

Now let's use this extension plus the new `Result` extension and finish our `CharacterProvider`. (commit: `3cd81be`)

```swift
public protocol CharacterProviding {
    func characters(_ done: @escaping Done<[Character], MarvelError>)
    func character(by id: Int, _ done: @escaping Done<[Character], MarvelError>)
}

public final class CharacterProvider: CharacterProviding {
    
    private let client: HTTPPerforming
    
    init(client: HTTPPerforming) {
        self.client = client
    }
    
    public func characters(_ done: @escaping (Result<[Character], MarvelError>) -> Void) {
        client.perform(Endpoint(path: "/v1/public/characters")) { (result: Result<Page<[Character]>, Error<MarvelError>>) in
            done(result.map(\.results).mapTerror(\.marvelError))
        }
    }
    
    public func character(by id: Int, _ done: @escaping (Result<[Character], MarvelError>) -> Void) {
        client.perform(Endpoint(path: "/v1/public/characters/\(id)")) { (result: Result<Page<[Character]>, Error<MarvelError>>) in
            done(result.map(\.results).mapTerror(\.marvelError))
        }
    }
    
}
```

And bomb, we find out a big problem. We cannot compile the app. `DataProvidersKit` it's so coupled to our `MarvelClient` so wee need to keep refactoring to make our app work again.

When doing frameworks, we must try to make them as less coupled as possible, my idea of frameworks is that them could be shareable, where we can take one out and put another and keep our software still working ([liskov substitution principle](https://stackify.com/solid-design-liskov-substitution-principle/)). 

Making frameworks with so closed dependencies, maybe we're only adding the extra complexity of have two frameworks, but we're not winnig anything more than that. **Complexity**.

## Conclusions

We splited one module doing to much things into two new framworks, one of them it's a library that could be shared along all the company and open sourced if we want.

We reduced functions that have a long body with new small pieces of code, that we could compose them to make more with less.

We gain the hability to parse the data that returns the server inside the `httpbody` of the response in a generic way. Making this help us to combine some cases where we need an `HTTPCode` and the body of the response. 

We reduced the potentially classes number inside our `MarverAPI` by transofming the `Request` creation from one base class to one function, and making clearer on our `Provider` what endpoint are using. Also taking the version of the endpoint out of the host, becuase one API could update one endpoints to the next version and others not, we keep that flexibility.

Our client gained a way to authroize request by a protocol, again, one small component, that give us a lot of flexibility.

Now we can mock our client with painless since its decoupled by the protocol `HTTPPerforming`, allowing us to provide a mock client that give us stubs of the responses.

On the next chapter, we will refactor the `DataProvidersKit` to make the app work great again. Also, we're going to make dependency inversion between `Repositories`(New `ProvidersKitName`) and the `MarvelClient`. We will move the `CharacterProviding` protocol to `Repositories` module, and then, we will have the needed flexibility to perform refactors on the future, as new techniques arrive, or we need to add extra functionality.

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
