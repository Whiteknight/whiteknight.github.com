---
layout: post
categories: [projects, resolution2017, acquaintance]
title: Acquaintance
---

This year instead of coming up with one rediculous and implausible resolution, I came up with a handful of goals. One of these was the goal to work on a new software project every month, either starting something new or bringing something old to a point where it can be released and used by others. In the best case I finish the year with several public projects to my name, and in the worst case I leave a trail of a dozen half-assed incomplete messes. Only time will tell.

For the month of January I decided to bring my library **Acquaintance**, which I started late in 2016, up to a 1.0 release. As of this writing, [version 1.0.0-rc1 is released on nuget](https://www.nuget.org/packages/Acquaintance). This will become 1.0.0 when I can get a bit more real-world testing on it.

## What is Acquaintance?

Acquaintance is an intra-process messaging library. It is designed for several use cases, among them:

1) To facilitate late-bound communication between modules in a loosely-coupled system
2) To help to isolate and abstract dependencies
3) To help with problems relating to thread synchronization and access to thread-unsafe resources

I really suggest you [check out the site](http://whiteknight.github.io/Acquaintance) or
[browse through the repository on github](http://github.com/Whiteknight/Acquaintance) to get a better sense of what this library is, what it does, and how you can use it in your own projects.

Acquaintaince started from a personal need while developing another piece of software. I was looking for an in-app messaging solution so I initially looked at [Postal.NET](https://github.com/rjperes/Postal.NET) (a .NET port of the popular [Postal.JS library](https://github.com/postaljs/postal.js)). That library didn't quite meet my needs, so I started to look elsewhere and found...nothing.

Several years ago I was developing a Silverlight application using the [Microsoft PRISM guidance for WPF](https://msdn.microsoft.com/en-us/library/ff648612.aspx). In that library they had implemented an interface `IEventAggregator` which performed a similar function: pub/sub messaging for loosely-coupled applications. I liked the idea pretty much back then, and have implemented and re-implemented the idea several times in several situations. This time around, I realized that the core of what I was trying to do could be more than just another throw-away implementation, but could instead stand alone as a reusable library. Acquaintance was born.

## Patterns

Unlike the `IEventAggregator` from PRISM, Acquaintance implements several messaging patterns: publish/subscribe, request/response and scatter/gather. Unlike some other options such as Postal.NET, These other patterns are built into the core of the library and aren't extensions built on top of publish/subscribe.

In addition to these three core patterns, Acquaintance also offers a few other important features such as "eavesdropping" (monitoring for request/response events), unit testing with mocked channels and expectation verifications, and processing networks.

## Learn more

The [github site](http://whiteknight.github.io/Acquaintance) has plenty of discussion and code-examples which I won't duplicate here. In the coming months I'm also going to be posting about projects which use Acquaintance internally.
