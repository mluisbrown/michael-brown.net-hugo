---
title: "Using PHLivePhotoView with Auto Layout"
date: 2016-11-03
draft: false
tags:
- swift
- dev
- autolayout
---

So, I was working on updating my app [Memories](https://michael-brown.net/memories/) to support displaying Live Photos properly and not just as a static image. I took a look at the documentation for [`PHLivePhotoView`](https://developer.apple.com/reference/photosui/phlivephotoview) and thought that this would be fairly straightforward. For Live Photos, I'll just use a `PHLivePhotoView` instead of a `UIImageView`.

Well, as is usually the case when a developer says: "this looks straightforward, I'll have it done in a couple of hours", it was not quite so straightforward and had me scratching my head and cursing until 2am until I finally realised what I was doing wrong.

First I wasted an hour or so on a diversion into trying to use Swift 3 generics for implementing a view that could display either a `UIImageView` or a `PHLivePhotoView`, and failing miserably. But that's a topic for a different post. Once I got back on track I quickly had the Live Photo view solution implemented. But all I got was a blank screen. The view was invisible. It didn't even show up in Xcode's view debugger. For normal photos using a `UIImageView` it was all working fine and the only difference for live photos is that the `UIImageView` and its `UIImage` is substituted by a `PHLivePhotoView` with its associated `PHLivePhoto`.

In order that the photos can be zoomed and panned by the user the image view is inside a `UIScrollView`. When you're using auto layout as long as you pin the edges of your view to the edges of the scroll view you never have to set the scroll view `contentSize` and everything "just works", as explained in Apple's [Technical Note TN2154](https://developer.apple.com/library/content/technotes/tn2154/_index.html#//apple_ref/doc/uid/DTS40013309-CH1-TNTAG3).

I checked my autolayout constraints in runtime over and over again, everything looked correct and exactly like it is when using a regular `UIImageView`. But then when I was using the Xcode view debugger I saw a tiny warning that said: "scrollable content size is ambiguous". The scroll view auto layout approach mentioned above works with `UIImageView` because `UIImageView` returns the size of its `image.size` for its `intrinsicContentSize`. The TN above even says so: "A simple example would be a large image view, which has an intrinsic content size derived from the size of the image".

But surely `PHLivePhotoView` also returns its `livePhoto.size` for `intrinsicContentSize`? Well, no, it doesn't. It returns a `CGSize` with each dimension as `-1`. So that explains it. The scroll view has no way of determining the correct content size. The solution is simply to add width and height auto layout constraints to the `PHLivePhotoView` equal to the `size.width` and `size.height` of the live photo view's `livePhoto`. Problem solved. All works perfectly now.

I cannot think of any reason why `PHLivePhotoView` would not behave as `UIImageView` does and return the size of its underlying image for `intrinsicContentSize`. Once I've filed a Radar I'll update this post with it's ID.