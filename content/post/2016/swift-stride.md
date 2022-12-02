---
title: "Using stride to convert C-style for loops to Swift 2.2"
date: 2016-03-08
tags: 
- swift
- dev
---

With the release of Swift 2.2 in Xcode 7.3 C-style for loops have become [deprecated](https://github.com/apple/swift-evolution/blob/master/proposals/0007-remove-c-style-for-loops.md). The default Xcode fix-it for converting them uses a Range:

```swift
let count = 5
for var index = 0; index < count; index++ {
    doSomething(index)
}
```

is converted do:

```swift
let count = 5
for index in 0 ..< count {
    doSomething(index)
}
```

For the vast majority of C-style for loops this will work perfectly well. But there are cases where using a Range not only won't work as expected but will actually crash! 

Consider the case where `count` in the above example is -1 (we'll get to why you might do that in a second). The C style loop would finish immediately as `index` is already greater than `count` on entry. However, the conversion using a Range would crash with:  `fatal error: Can't form Range with end < start`.

So, why on earth might you want have a loop range with an end less than the start? In my app [Memories](https://memories.land) (code available on [GitHub](https://github.com/mluisbrown/Memories)) I have a View Controller which is an image viewer where you can swipe left and right to navigate a set of images. In order to save memory, I only want to have the current visible image and the ones immediately to the left and to the right of it loaded in memory. As the user swipes through the images, all other images are purged from memory.

To purge all the images to the left of the visible image I had a loop that looked like this:

```swift
let page = visiblePage
let firstPage = page - 1
let lastPage = page + 1

// Purge anything before the first page
for var index = 0; index < firstPage; ++index {
    purgePage(index)
}
```

If the currently visible image is the first one (`page == 0`), then `firstPage` will be -1. In the Range version of the loop:

```swift
for index in 0 ..< firstPage {
    purgePage(index)
}
```

we end up with a Range where the end is less than the start. Boom. Ok, this only happens in the edge case where we are on the first page (and the last page, in a similar case where the start is greater than the end). We could wrap the loop in an `if` that only executes the loop when `page > 0`. But I really hate writing special logic for edge cases if I can possibly avoid it. It's so much cleaner if your logic handles edge cases by default. So what's the solution? The Swift [`Strideable`](https://swiftdoc.org/v2.1/protocol/Strideable) Protocol, particularly the [`stride(to:by:)`](https://swiftdoc.org/v2.1/protocol/Strideable/#func--stride-to_by_) method. As the docs state:

> Return the sequence of values (self, self + stride, self + stride + stride, ... last) where last is the last value in the progression that is less than end.

So, for example: `0.stride(to: 5, by: 1)` returns the sequence (0, 1, 2, 3, 4). Importantly, it doesn't matter if the the value passed in the `to:` parameter is less than the `self` (the start value) so, for example: `0.stride(to: -1, by: 1)` returns an empty sequence. So now we can just replace the Range with the sequence returned from `stride(to:by:)`:

```swift
for index in 0.stride(to: firstPage, by: 1) {
    purgePage(index)
}
```

the loop to purge the pages *after* the current one is similar, using a reverse sequence (`pages` here is the array of pages):

```swift
for index in pages.indices.last?.stride(to: lastPage, by: -1) {
    purgePage(index)
}
```

Now that's a lot nicer. No special logic for edge cases!

We can still improve on this though and make these loops more 'functional' and perhaps more idiomatic Swift using [`forEach(_:)`](https://swiftdoc.org/v2.1/protocol/SequenceType/#func-foreach_):

```swift
0.stride(to: firstPage, by: 1).forEach(purgePage)
pages.indices.last?.stride(to: lastPage, by: -1).forEach(purgePage)
```

Now that's looking really elegant. I'm tempted to use `stride(to:by:).forEach(_:)` to replace all my C style for loops in Swift.  There is a caveat though: you cannot break out of or continue a `forEach` iteration with `break` or `continue`. For those cases just use a `stride(to:by:)` sequence with `for index in`.

Thanks to [@NSRenato](https://twitter.com/NSRenato) for the tip on using `stride`.