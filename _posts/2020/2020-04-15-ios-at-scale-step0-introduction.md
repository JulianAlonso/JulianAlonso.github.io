---
layout: post
title: iOS at scale - Introduction
author: juli
---

Since the beginning of time, work on projects with big teams are hard. We need to be more organized, our processes must be well defined, all the team must follow the same rules, and that only talking about mindset. What about the code?

In our daily basis we can find ourselves spending hours fixing git conflicts, doing code that our mate already done, modifing a piece of code that other it's modifing on that moment, we may have different coding styles... 

There're some problems like implement two new functionalities on the same file that we can't avoid, thats a fact. But we can reduce a lot how many conflicts we will have, we can remove `xcodeproj` related conflicts, we can stablish a guide for code styling, we can reuse so many code as we can, starting with sharing our libraries along the whole company, and of course, we can reduce our code quantity, at least having the same flexibility, make it easy to read, easy to understand and harder to introduce bugs.

All these lines sounds like the impossible goal to archieve, but in these post series, we're going to see **step by step** how some new practices, tools and approaches could be applied on our daily basis to make all of this a reality.

## Proposal

In this post I'll try to cover how we can create an easy project setup for an iOS app, that Scale on a **hugh** company with **big** teams.

To archieve this, we're going to take a project, refactor it step by step, showing up how a couple of new tools and new approaches that could help us to reduce complexity, avoid git conflicts, write less code, be cleaner, faster and of course, improve the code testability.

Also I'm going to introcue a little concepts of functional programming, trying to make us lose the fear about [FPP](https://www.geeksforgeeks.org/functional-programming-paradigm/), and how these modern approaches could make our more composable and flexible.

To do this, I'm going to try avoid the usage of many external libraries as I can, explaining on every point what I'm doing and why, and why I guess it's a good choice to scale a project. This not only may help you on big projects, but also may help on projects with one or two developers. Remember that always is a good option keep your project and code as Simple as posible.

Note that the tools that I'm going to use here are not the uniques on it's `scope` and you'll find others that could help on those tasks. 

## Why

With the quantity of information on internet, nowadays could be hard for us find what it's a good solution or not. We've find a lot of "you must do this" or "you must do that", "you're coupling that part" or whatever, but isn't usual to find out a post series that given a project performs a refactor and justify **every choice taken about how and why**.

This post series has born with the idea of cover up how a project based on minimum verision on `iOS 10`, could scale, avoiding the most usual problems on big and not so big teams, how we can make our life as developers easier, our code more flexible to change without not so much effort, and cover how new techniques help us coding.

We've going to start on a [project](https://github.com/JulianAlonso/excelsior) provided by [Rafael Bartolome](https://twitter.com/RafaelBartolome) where we have a very good start point, due that the project it's talking about iOS at Scale, and we will discover post by post, step by step, commit by commit, why this project could scale even better and why.

Okey, given this, we're going to start clonning the same project from [this github Project](https://github.com/JulianAlonso/excelsior), were we have all the step refactors done, tagged and we will be lead by the posts.

These posts are lead by steps, these steps are on the project, so I suggest you clone the project if you didn't did it before, go to the `step0` (`git checkout step-0`), take a look, and then follow each post while you see how the code changes, **keeps compiling _almost_ on every step** and **understading** what are we doing and why.

Take into account that this is an opinionated refactor, based on my previous experience and how I've end up building software after some years of career. So maybe you'll find some ideas or tools that doesn't fit with you or your needs at all, so please, read this with open mind, say welcome to changes and enjoy!

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
