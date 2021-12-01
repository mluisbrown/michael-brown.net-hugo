---
title: "The future of The ReactiveSwift Composable Architecture"
date: 2021-12-01
tags:
- swift
- dev
---

## Introduction

The [ReactiveSwift Composable Architecture](https://github.com/trading-point/reactiveswift-composable-architecture) is a fork of the [Point-Free](https://github.com/pointfreeco) [Composable Architecture](https://github.com/pointfreeco/swift-composable-architecture). It uses [ReactiveSwift](https://github.com/ReactiveCocoa/ReactiveSwift) as the basis for the `Effect` type instead of Apple's Combine framework.

The reason why I created the fork, in May 2020, was because Combine is only available on iOS 13 and macOS 10.15 and later. At the time I was working on a brand new iOS app where I really wanted to use TCA but we couldn't yet set a minimum deployment target of iOS 13. I was sure there were others who also were in the same position: not being able to use TCA because of the iOS 13 requirement.

Soon after the fork was released I got requests to make it compatible with Linux. There was someone wanting to use a Linux compatible version of TCA on an Android project using Swift with the [Swift Android Toolchain](https://github.com/readdle/swift-android-toolchain)! It turns out that it wasn't that hard to make TCA work on Linux too (obviously without any of the UI components).

## Time marches on

For the past 18 months I have maintained the fork in sync with TCA: adopting all the new features and changes and keeping the release versions pretty much in sync. At the same time, the project I was working on, a brand new FX Trading app for [Trading Point](https://www.trading-point.com), was getting close to its first production release. iOS 15 had already been released and iOS 14 was a year old. We decided to up our minimum deployment target to iOS 13. It was also decided to migrate the code to use the original TCA and Combine, and no longer use the ReactiveSwift port.

The migration from ReactiveSwift TCA to Combine TCA was surprisingly straight forward, and I know of other projects that have done the same thing. This actually shows how well designed TCA is, that it is sufficiently well abstracted from the underlying Functional Reactive Programming framework.

Shortly afterwards, I also left Trading Point and have started a new role at [Argent](https://www.argent.xyz), where we are using Combine TCA in the iOS app.

[Brandon Williams](https://github.com/mbrandonw) and [Stephen Celis](https://github.com/stephencelis) continued developing TCA with new features and improvements. Some of these use libraries that they have developed, such as [Swift Identified Collections](https://github.com/pointfreeco/swift-identified-collections) and [Swift Custom Dump](https://github.com/pointfreeco/swift-custom-dump) which also require iOS 13 or macOS 10.15 as a minimum. This has made maintaining the fork increasingly difficult, as the differences are now not just the FRP library, but the fork also has to maintain older code that doesn't use these new libraries.

Lastly, Swift 5.5 brought new concurrency features to Swift such as async/await, Actors, Tasks and AsyncSequence, and from Xcode 13.2 onwards, these features have [been backported to iOS 13 and macOS 10.15](https://forums.swift.org/t/swift-concurrency-back-deployment/51908/9). This raises the possibility that TCA itself might choose to migrate away from using Combine (a closed source Apple only framework), to using Swift concurrency features (which are open source and cross platform) for modelling the `Effect` type.

## The future

I'm unsure what the future of the ReactiveSwift fork of TCA is. There are several factors involved:

1. It has become harder to maintain, particularly if compatibility with iOS 11 & 12 is kept.
2. It is no longer as relevant or necessary as it was.
3. I'm no longer working professionally on a project that is using it.
4. There is the potential (just speculation on my part), that a switch to Swift Concurrency might happen.

As a first step, I've taken the decision that release [0.28.1](https://github.com/trading-point/reactiveswift-composable-architecture/releases/tag/0.28.1) of the ReactiveSwift fork will be the last to support iOS versions below iOS 13 and macOS versions below macOS 10.15. From the next release on, the minimum targets will be iOS 13 and macOS 10.15. This will considerably reduce the maintenance burden.

Over the next few weeks and months I will continue to evaluate whether it makes sense to maintain the fork at all.

Lastly, I want to give a huge thank you to Trading Point / XM and all mobile team there. It was a pleasure working with you all ❤️ 🙏
