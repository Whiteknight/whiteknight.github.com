---
layout: post
categories: [Parrot, Rosella, Release]
title: Rosella Release 1
---

Yesterday [I committed and tagged][release_commit] the first release of
[Rosella][]. Rosella is a collection of libraries to help enable the use of
common patterns and best practices in Parrot.

[release_commit]: https://github.com/Whiteknight/Rosella/commit/0ab2e93053909912aa48790686ea5c774d5006cc
[Rosella]: http://Whiteknight.github.com/Rosella

The Rosella release targets the 3.3 version of Parrot, however future releases
will not target every supported release of Parrot. Instead, I will cut a
Rosella release when we've had enough changes and new additions to warrant
one. Rosella releases will always target a stable Parrot release.

This first "official" version of Rosella is in many ways a prototype. Besides
some of the testing tools, I am not aware of anybody who is using Rosella in
their projects. I am very interested in hearing suggestings about new
features, reports of bugs or problems, and any other feedback that people may
have about the project or its component libraries. Nothing is set in stone
with Rosella, and if I need to make semantic changes or API changes to better
suit the needs of the users, I will gladly do that.

Also, contributions are always welcome!

The first Rosella release ships with 9 stable libraries:

1. **Core**: The central library that implements standard constructors, among
   other things.
2. **Action**:  A library to implement Actions, which are similar to the
   Command pattern.
3. **Container**: A dependency injection, inversion of control container
   library.
4. **Event**: An implementation of the publish/subscribe pattern
5. **Proxy**: A library for implementing and constructing custom-purpose
   proxy objects.
6. **Winxed**: A library for porting some common build infrastructure to
   Winxed.
7. **Test**: A library for xUnit-style unit testing
8. **Harness**: A library for implementing a TAP harness
9. **MockObject**: An implementation of mock objects for Parrot.

See the Rosella website for more details and code examples for each of these.

In addition, the project ships with a handful of unstable prototype libraries:

1. **Query**: A library of higher-order functions, similar to System.Linq
   from C#.
2. **Path**: A library for traversing nested aggregates by path strings.
3. **Contract**: A library for adding assertions and contract-like behaviors
   to programs.
4. **Decorate**: An implementation of the decorator pattern.
5. **Memoize**: A library to cache results from common functions.

Here, "unstable" means that these libraries should be usable for some
situations, but are undocumented, untested, incomplete, and mostly available
as a curiosity. If you want more information about any of these unstable
libraries, or have feedback about them, I am always happy to chat about it.

I don't have any plans or estimates for the next Rosella release. It could
happen at any time when I think Rosella has made enough progress to warrant
one. Of course, since I've put in the effort to upgrade the build
infrastructure now, it's absurdly easy to make a new release when I decide
it is time.
