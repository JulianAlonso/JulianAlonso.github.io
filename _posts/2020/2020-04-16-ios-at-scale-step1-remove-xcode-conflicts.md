---
layout: post
title: Xcode at scale - Xcode at scale
author: juli
---

This is the first post of a serie named **iOS at Scale** based on the next steps:
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


If you already have this type of conflicts removed from your project, splited your project in modules (with any project generator), you may go to the [STEP 2]({% link _posts/2020/2020-04-17-ios-at-scale-step2-refactor-marvel-client.md %}).

Based on the given Architecture, we can see the benefits of have our app splited in several Modules. We have forlders with code linked to a namespace, using the `public`/`private`/`interal` modifiers with sense and creating different spaces for each team of developers or developer.

The first thing we're going to do to allow our project scale without problems, it's to reduce as many as we can `xcodeproj` conflicts. This conflicts are the main base of pain when working on huge (and not so huge) teams. This conflicts are mainly produced by the simple reason of adding a file on two different branches. To avoid the usage of _tricks_ and waste time solving conflicts that we can get out we are going to remove as many `xcodeprojs` as possible (all of them).

This is a very common problem on our days, Xcode configuration and Xcode compile times at Scale are known as not the best option, companies like Facebook or Google have developed other compilers ([Buck](https://buck.build/), [Bazel](https://bazel.build/)) with features like project generation, and reducing compile times. Those tools could be hard to setup, and if we don't need super fast compile times, we're going to try to remove at least the `xcodeproj` conflicts using an  provided tool, [Swift Package Manager](https://swift.org/package-manager/).

[SPM](https://swift.org/package-manager/) (and any dependency manager tool ([cocoapods](), [carthage]()...)) allows us to create small, reusables libraries (we have the limitation to build only static libraries with SPM). We not only are going to remove some `xcodeproj` pain, we are winning the option to share our small libraries along all our company. Have our libraries like the `http client` shared, allows us to versioning our library, share the same DSL along our developers, and solve bugs in one time for the whole company.
This also give us the option, if we want or our company allow us to, the hability to open source it.

To define our library as an SPM package, we only need to create a `Manifest` file, called `Package.swift` and specify where the code and the test are, along with the product that it builds. As any other dependency manager we can tag to specified version (to avoid the need of refactor code if we don't have time), and gain the option to share it not only on iOs, also on Mac, TV and Watch apps.

So, I'm going to transform all the project modules into SPM, add it as dependencies to our App projects and then, create a Workspace that allow us to work on all of them at the same time. 

Note: On big teams, maybe shared libraries it's not the best option to be edited by the whole company, could be a good option have them in other repository and work on them when needed.

### SPM Steps.

All of this steps are under git commits, so you can see all the sub steps that I do.

The first thing I did its move all the modules of the app to folders, create the packages (not added dependencies yet), we've archived to have a cleaner folder project, but we keep all the modules under `excelsior` folder, that way **it's not so clean**. The next step, it's to move all this modules under a `Module` folder, this way, with a simple look to the project on github or the folder inside your system, we may see the project structure fast. 

If you go to the commit `15e96ed` (git checkout 15e96ed), you will se how clean it's currently the project folder. This is extreamly useful when wee see the code on github for the first time, and help us to know how the code is distributed. Also, allow us to take a folder inside the project and in a few steps we have a separated library, ready for reuse (take care of dependencies). So, now, we only reorganized code into more illustrative folders. 

With the `Manifest` created, it's time to link the projects each other, and try to make the app compile again.

Due to I'm not the creator of the project, I'll create every project with `swift package generate-xcodeproj` and checking its dependencies to link needed libraries.


...


After some time trying to do this, we will find that SPM doesn't fits well for this use case.

This project has view in `.xibs` and `.storyboards` and resources like images, since SPM only builds static libraries, we can't add any more than code. If we don't do all our views by code ([SwiftUI](https://developer.apple.com/xcode/swiftui/), [Texture](https://texturegroup.org/)) and we don't have images, SPM doesn't not Support our Scale (hopefully yet). So we need findout a good solution to cover our use case.

But we've gained one main thing. **Cleaner folder organization**. And yes, this will help us a lot on the next solution.

We need a tool that avoid `xcodeproj` conflicts, allow us to have different modules, with different dependencies, have different Manifest files for each of our projects and if it could be possible written in swift.

Do you know what tool we need right?


### Tuist

[Tuist](tuist.io) its a tool that allow us to make our project by modules, generate the xcode on the fly based on Manifest files, and so much more that you cand find on the [docs](https://tuist.io/docs/usage/getting-started/).

We're going to create the needed manifest files called `Project.swift`. Note that Tuist also offer us `ProjectDefinitionHelpers` that allow us to extend his code base to add functions and simplify our configuration. Making our modules following the same structure that [SPM follows](https://swift.org/package-manager/) under `Creating a Library Package`, will simplify all our configuration in a very easy and _understanding_ way.

First, lest create our ProjectDefinitionHelper` for all our modules, specifing the main target and the test target. This files must be under `Tuist/ProjectDefinitionHelper`.

Our helper to create modules will look like this (you can find it on commit `4645d6b`).

```swift

extension Project {
    public static func module(name: String, dependencies: [TargetDependency] = []) -> Project {
        Project(name: name,
                targets: [
                    Target(name: name,
                            platform: .iOS,
                            product: .framework,
                            bundleId: "com.excelsior.\(name)",
                            deploymentTarget: .iOS(targetVersion: "10.0", devices: .iphone),
                            infoPlist: .default,
                            sources: ["Sources/**"],
                            dependencies: dependencies),
                    Target(name: "\(name)UnitTests",
                            platform: platform,
                            product: .unitTests,
                            bundleId: "com.excelsior.\(name)Tests",
                            deploymentTarget: .iOS(targetVersion: "13.0", devices: .iphone),
                            infoPlist: .default,
                            sources: "Tests/**",
                            dependencies: [
                                .target(name: "\(name)"),
                            ])
                ])
    }
}


```

Next step.

We need to create the manifest file to all our projects.

Our `AppCore` manifest file will look like:

```swift 
import ProjectDescription
import ProjectDescriptionHelpers

let project = Project.module(name: "AppCore")
```

I think **more simpler it's impossible**. This will generate our `xcodeproj` on the fly `xcodeproj` without effort. (commit `94ecf97`).

Now let's generate the rest of the modules `Manifests`. You can find this done at `7ca2a96`

It's time to get out our App project `xcodeproj`. Our App project Manifest will look a little bit different, but not too much. In order to do that, I'll move all the App files under a folder called `App` so we again gain a cleaner files and folder structure.

All these changes can be found on commit `fd92561`.

Now I like to create a workspace with our app (all the project dependencies projects, not linked yet, will be generated also when they're linked).
The Manifest for the project has the name `Workspace.swift` and will look like this: (commit: `3161c97`)

```swift
import ProjectDescription

let workspace = Workspace(name: "Marvel",
                          projects: ["App"])
```

Okey, it's time to generate our project (command: `tuist generate`). After fix a little error on UITest sources declaration on `App/Project.swift` we end up with the Excelsior App project generated. hash `90e4180`.

Now its time to link our dependencies, before of do this, I'm going to create another `ProjectDefitionHelper` to name my dependencies and then have the hability to add them with the swift dot notation `.dependencyName`. This is extreamily useful, because we get abstracted of how our dependencies are provided. We don't care if the dependencies are given by `Projects`, `Carthage` o even `Packages`. _Note that project dependencies will be added to the workspace too and we can work on that while packages, carthage or cocoapods dependencies not_

Lets introduce here the most useful Tuist command. `tuist edit`. this command allow us to edit the project configuration files **inside xcode** (more info [here](https://tuist.io/docs/commands/edit/)). So we will run `tuist edit` under `excelsior` folder and it will open up the workspace with the project definition helper folder (what it's the folder that interest us now).

Our dependencies helper will look like: (commit: `7a37167`)

```swift
import ProjectDescription

public extension TargetDependency {
    // App
    static let appCore: TargetDependency = .project(target: "AppCore", path: .relativeToRoot("Modules/AppCore"))
    // Features
    static let characterDetailKit: TargetDependency = .project(target: "CharacterDetailKit", path: .relativeToRoot("Modules/CharacterDetailKit"))
    static let characterListKit: TargetDependency = .project(target: "CharacterListKit", path: .relativeToRoot("Modules/CharacterListKit"))
    // Core Dependencies
    static let dataProvidersKit: TargetDependency = .project(target: "DataProvidersKit", path: .relativeToRoot("Modules/DataProvidersKit"))
    static let displayKit: TargetDependency = .project(target: "DisplayKit", path: .relativeToRoot("Modules/DisplayKit"))
    static let marvelClient: TargetDependency = .project(target: "MarvelClient", path: .relativeToRoot("Modules/MarvelClient"))
    static let navigatorKit: TargetDependency = .project(target: "NavigatorKit", path: .relativeToRoot("Modules/NavigatorKit"))
    static let support: TargetDependency = .project(target: "Support", path: .relativeToRoot("Modules/Support"))
}
```

Now let's link our dependencies. To do this, I'm going to start from the App target, trying to compile it, and then going resolving dependencies.
Out app don't compile why we dont have AppCoreKit dependency. 

At this point I'll add some notes:
- _I've removed the suffix kit on the organizing project so now we need AppCore_
- _I'll be renaming things only to fell more confortable_
- _On code steps, I'll rename some things in order to follow industry standards, and I'll add documentation about why when I do it_

![Failing dependency]({{ basepath }}/resources/images/posts/ios_at_scale/app_core_kit_failing_dependency.png){: .img-rounded .img-responsive }

We change the AppCoreKit to AppCore and add the dependency to the `App/Project.swift` with `dependencies: [.appCore]`. After generating the project we will find out that our Workspace now have a Group called `Modules` with our `AppCore` inside (hash `4f2ef3b`).

![Workspace with app core]({{ basepath }}/resources/images/posts/ios_at_scale/workspace_with_appcore.png){: .img-rounded .img-responsive }

Now, like this, I'll resolve all dependencies and try to compile the whole app. Note: _I previously deleted necesary Headers files, in the next commit you will see them in again._

Finally, after link all dependencies, I've added `ResourceType` to Tuist ProjectDefinitionHelpers, check that the app compile, tests are passing and everything it's working again.

You can check the code at this commit `ccf6155`. And yes, Kingfisher it's commented, so it's time to add the dependency. And end with the project settup.

Then, after add Kingfisher by SPM to the DisplayKit (previously `CommonUIKit`), we got it available on all projects that have DisplayKit as dependency. 
Our project it's finally in what a think, a good initial state.

- Folders organized
- Removed xcodeproj conflicts, we don't need to use tricks anymore
- The only dependency given by SPM (but you can use whatever dependency manager you want)
- From github the code will provide us an easy way to know how the app it's defined
- Maintaining the whole workspace where we can edit our code

![Final project setup]({{ basepath }}/resources/images/posts/ios_at_scale/final_project_setup.png){: .img-rounded .img-responsive }

(commit: `040bd43`)

To end with this chaper, the last thing we're going to do it's remove the `xcodeproj` and `workspace` from the repository, and also we will ignore the Tuist Derived folder. Now the project could be easy generated by `tuist generate`. (commit: `a1f95eb`)

## Clonclusion

We've seen how SPM actually doesn't fits our need by the moment, but how some other tools like [Tuist](tuist.io) could help us removing conflicts and making our interactions with Xcode easier. 

We have all the Xcode realted configurations inside swift code, and not inside the xcode project.

Also we've done a simple folder organization to have a clearer way about where is our code situated. Making us easy to understand how it's built.

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
