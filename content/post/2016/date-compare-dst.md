---
title: "Comparing Dates, whilst ignoring the time"
date: 2016-04-24
tags: 
- swift
- dev
---

Comparing dates is one of the most common things you have to do as a developer of almost any type of software. At first glance it would seem to be something almost trivially easy. What could possibly go wrong? Well, turns out, quite a lot!

I'm going to highlight just one issue that recently caused an embarassing bug in my app [Memories](http://memories.land). It involved just a simple date comparison, ignoring time, and without crossing timezones.

I had an array of `NSDate` and I wanted to get the index of the date in the array closest to the current date. The code was something like this:

<script src="https://gist.github.com/mluisbrown/8808c70899ed278e5e5e3ba06fee4320.js"></script>

The dates in the array have their time components all set to zero as I don't care about the time, I just want to compare the date. To do the actual date comparison so that the time component is ignored, I'm using the rather handy `NSCalendar` method [ordinalityOfUnit:inUnit:forDate:](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSCalendar_Class/index.html#//apple_ref/occ/instm/NSCalendar/ordinalityOfUnit:inUnit:forDate:). If you call this method with `.Day` for the `smaller` parameter and `.Era` for the `.Larger` parameter it will give you a sort of Julian Day number[^julian] that's easy to use for date arithmetic.

Looking at the code above, you would expect that the closest index when calling `let closest = indexOfClosestDate(someDates, toDate: NSDate())` would be 1, since today is closest to, well, today. However, if you run the above code in an Xcode Playground somewhere in the GMT timezone whilst DST is in effect you will see that the result is 2. The `NSDate()` for today is comparing as having an ordinal day number matching tomorrow, not today. What the hell is going on?

Well, the clue is in the phrase "somewhere in the GMT timezone whilst DST is in effect". It's one reason I didn't spot this bug in my app until fairly recently as I had done all the development in the winter when DST is not in effect, and I live in Lisbon in Portugal which is in the GMT timezone.

The following playground should make everything clear (note that the values shown in comments are values I obtained running in the GMT timezone with DST in effect late on April 24th 2016):

<script src="https://gist.github.com/mluisbrown/31146f28f3311e62345d5d4ef32be4ed.js"></script>

The ordinal day number for `today` is 736078. But the ordinal day number for `todayMidnight` is 736077. 1 day difference. But how is that possible, since they are the same day? Take a look at the `debugDescription` for each variable:

* `today`: "2016-04-24 21:16:39 +0000" 
* `todayMidnight`: "2016-04-23 23:00:00 +0000"

You see? `todayMidnight` is actually represented as the day *before* at 23:00, not today at midnight! The reason for this is that `NSDate` internally stores dates in [UTC](https://en.wikipedia.org/wiki/Coordinated_Universal_Time), which is the same as GMT for most purposes. When DST is in effect, the time where I live is one hour ahead of GMT, so today at midnight, in GMT terms, is actually yesterday at 23:00.

The solution I found for comparing dates where you don't care about the time component is to set the time component to midday (12:00) instead of midnight (00:00). That way, it would take a 12 hour offset due to DST or timezone changes to change the date in UTC terms.


[^julian]: Well, since I'm using the Gregorian calendar, it would be a Gregorian Day Number, which is called a [Rata Die](https://en.wikipedia.org/wiki/Rata_Die)

