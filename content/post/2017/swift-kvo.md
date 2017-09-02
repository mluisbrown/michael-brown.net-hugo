---
title: "Swift and KVO context variables"
date: 2017-09-02
tags: 
- swift
- dev
---

I came across a crash in my app [Memories](https://github.com/mluisbrown/Memories) when using the iOS 11 betas. It happened reliably every time I loaded a video, and the video code uses KVO, as `AVFoundation` requires you to use it for a lot of things.

When I ran the app in Xcode this was the exception that was causing the crash:

```
Simultaneous accesses to 0x1c41ded68, but modification requires exclusive access.
Previous access (a modification) started at addObserver(_:forKeyPath:options:context:)
Current access (a modification) started at observeValue(forKeyPath:of:change:context:)
```

The hex address was the address of the property I was using as the context passed to `addObserver()` and which I was checking in `observeValue`. I was using just a regular instance variable for the context. I was also using the `.initial` option when adding the observer in order to observe the initial value, not just changes:

<script src="https://gist.github.com/mluisbrown/e90f078200d3c88925d94e09a0e0594d.js"></script>

When you use the `.initial` option `observeValue(forKeyPath:)` is called immediately as part of the call to `addObserver()`, in the same stack frame. This is what leads the Swift runtime to believe that "Simultaneous access" is occuring to the context variable. Even though no actual modification is occuring, the runtime probably has no way to determine that for Swift pointers bridged to `UnsafeMutableRawPointer` and so is playing it safe.

I found the solution to the problem in this [Apple Swift Blog post](https://developer.apple.com/swift/blog/?id=6) about interacting with C pointers from July 2014! So this has been a problem right since the start of Swift, but only now, perhaps with improved runtime checks in Swift 4, is it causing crashes.

The relevant part is here (emphasis mine):

> These conversions cannot safely be used if the callee saves the pointer value for use after it returns. The pointer that results from these conversions is only guaranteed to be valid for the duration of a call. Even if you pass the same variable, array, or string as multiple pointer arguments, you could receive a different pointer each time. **An exception to this is global or static stored variables. You can safely use the address of a global variable as a persistent unique pointer value, e.g.: as a KVO context parameter.**

The "conversions" referred to above are the bridging of `&varName` to an `UnsafeMutableRawPointer`.

So the solution is very simple. Either pull the context variable out of the class and make it a global variable (which can still be `private` so it's not visible outside of the file in which it's declared), or make it a `static` class variable:

<script src="https://gist.github.com/mluisbrown/3c91d8e07af52aebfca9e897fad99e6d.js"></script>

There are lots of bits of KVO sample code on Stack Overflow and elsewhere which don't follow this rule, so a lot of people are going to get burned with Swift 4, as the solution is not well documented, or easy to find.

On the other hand, this problem only occurs if you use the `.initial` observing option and if you use a context variable, which you [should](https://stackoverflow.com/a/11917449/368085), although many people don't.