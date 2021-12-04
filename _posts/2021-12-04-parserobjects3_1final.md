---
layout: post
categories: [projects]
title: ParserObjects 3.1 Released
---

At some point you just have to pull the band-aid off. The perfect is the enemy of the good, and at some point I have to decide to either keep fiddling with it indefinitely or just publish something. Today I published something. I'm happy to announce the release of [ParserObjects v3.1](https://www.nuget.org/packages/ParserObjects/3.1.0).

As I mentioned in [a previous post](http://whiteknight.github.io/2021/09/12/parserobjects3_1.html) the big feature of this release is the addition of the [Earley parser](http://whiteknight.github.io/2021/01/09/earleyalgorithm.html) and the new concepts of multi-parser and multi-result. Since my last post a lot of work has gone into integrating the `IMultiParser` and `IMultiResult` abstractions into the core of the library to have feature parity with other types of parsers. In this post I'll be talking about some features I didn't mention previously which made it into the release, and I'll also be going a bit more in-depth about multi-parsers.

## On Breaking Changes

Though this is a minor version bump and I generally adhere to *Semantic Versioning* rules here, there are a small handful of breaking changes related to interface members and class names/organization. I have updated the documentation to show that referencing the parsers directly by class name is not really a supported or encouraged use-case, and I don't expect that most people were doing it anyway. If you are a power-user who is using lots of the nooks-and-crannies of this library, please upgrade from 3.0 to 3.1 with care.

I had several other changes ready to go which I deemed to be "too much breakage" so I backed those out and am scheduling them for a 4.0 release eventually. The few breakages that remained were deemed to be so small and so essential that it was worthwhile keeping it as 3.1. 

The thing is, if I just decided to number this release "4.0" then there would have been a lot of other changes I wanted to also make, and the release cycle would essentially be unlimited length. I had to draw the line somewhere. 

## MultiParsers and MultiResults

First, let's start with a note:

**Note**: I don't know if there is a more appropriate term in the literature to separate a parser which returns a single parse tree from one that returns an entire parse forest. In lieu of a more established term, I'm using the terms "single-parser" (which I often simplify to just "parser") and "multi-parser" to differentiate the two. These two concepts are implemented with the interfaces `IParser<TInput, TOutput>` and `IMultiParser<TInput, TOutput>` respectively.

The Earley algorithm returns multiple results in the face of ambiguous grammars. If there are more than one way to parse the given input, Earley will return each of them. Often this ambiguity is undesireable, though there are some cases where we may want to intentionally allow multiple paths of progress, or provide multiple views and interpretations of the input data. With this in mind, I've taken some considerable effort to add a lot of infrastructure around multi-parsers so they can be used as the parser designer sees fit.

### Some Basics

`ProduceMulti()` is a new parser similar to the `Produce` parser but instead of returning a single value it returns many possible values.

```csharp
var produce = ProduceMulti(() => new[] { 'A', 'B', 'C' });
```

Most of the time I like to try to keep names the same between single-parser and multi-parser variants of the same thing, but in this case the two versions of `Produce()` cannot be differentiated just by the type system alone. 

```csharp
var produce = Produce(() => new[] { 'A', 'B', 'C' });
```

The first example is a multi-parser which returns possible results of type `char`, while the second is a single-parser which returns an array of type `char[]`. This is why I needed the methods to have two different names. In addition to `Produce` many other existing parsers are extended to work with multi-parsers as well:

```csharp
var parser = Deferred(() => 
    Try(
        Replaceable(
            Cached(
                ProduceMulti(() => ...)
            )
        )
    )
)
```

All of these work with multi-parsers exactly the way you expect if you've used the single-parser variants.

### `Each`

The `Each` parser takes a list of single parsers and executes them all at the current position, returning all the individual results as a single multi-result from which we can continue parsing.

```csharp
var parser = Each(
    p1, 
    p2,
    p3
)
```

### `ContinueWith`

Next up is `.ContinueWith()`. This parser is conceptually similar to the existing `.LeftApply()` parser, and also shares a lot of ideas with the `.Select()` and `.SelectMany()` LINQ extension methods. It's not exactly a *Functor*, I'm just saying that there are some conceptual similarities between these.

The `.ContinueWith()` method allows composing the initial multi-parser as if it were a single-parser, and continues the parse forward with each successful possibility in parallel. Here's a quick example:

```csharp
var initialValues = ProduceMulti(() => new[] { 'A', 'B', 'C' });
var parser = initialValues.ContinueWith(left =>
    Rule(
        Produce(() => '(')
        left,
        Produce(() => ')'),
        (open, l, close) => $"{open}{l}{close}"
    )
);
```

What we're doing in this example is starting with a few values `'A', 'B', 'C'` and wrapping each one in parenthesis. The interesting bit is the `left` symbol in the callback, which is a synthetic `IParser` that represents each value from the `initialValues` parser, one at a time. The output of this is another multi-result which contains the list of parenthesized characters.

`.ContinueWith()` is powerful because it allows us to start with an ambiguous prefix and continue parsing with each possibility. Some of the possible routes will, hopefully, fail and the system will pare down to a single unambiguous result eventually.

### `Transform`

The `.Transform()` multi-parser is very similar to a combination of `.ContinueWith()` and the `Transform` single-parser. Basically you take a multi-result, transform each of the values in it, and return a new multi-result with the transformed values. 

### `Select`

The `.Select()` parser and it's pseudonyms is extremely important because it allows us to integrate a multi-parser into a graph of single-parsers. `.Select()` allows us to choose a single result from the resulting parse forest to use going forward. 

**Note**: I'm not happy with the naming of this one and it's likely to change in the next major release. While the method is literally selecting a single result, I don't like how it conflicts with the `.Select()` LINQ extension method. I do have other parsers which conflict with names of LINQ methods, such as `.First()`, though `.First()` parser operates quite similarly to the `.First()` LINQ method: check each item and return the first one that matches a predicate (with the "predicate" here being "returns success"). Here, the `.Select()` parser actually has behavior more similar to `.Where()` or `.FirstOrDefault()`. Since it's basically a catamorphism we could call it `.Aggregate` to match with LINQ, but that doesn't seem to fit here. The next obvious name would be `.Choose()` though I already have a parser named `.Choose()`. Maybe I'll rename it to something like `.Pick()` or `.SelectOne()` or something like that in the future.

The `.Select()` parser takes the multi result and factories to create a single result from one of them:

```csharp
var parser = initialParser.Select((result, success, failure) => {

    // Return success() if we find a result we like
    if (result.Results.Any())
        return success(result.Results[0]);

    // return failure() if we can't find one we like
    return failure();
});
```

This `.Select()` parser is used internally to implement a few standard selectors like `.First()` to get the first result whatever it may be, `.Longest()` to implement a greedy algorithm taking whichever result consumed the most input, and `.Single()` which expects exactly one result.

## Caching

In addition to the changes mentioned in my previous post, I have adding a caching/memoization feature to the library. I know I said that it was something that I wasn't going to get to this cycle, but I found a simple way to do what I wanted and having caches around was a major performance improvement for production rules in the Earley parser. If you expect a parse to traverse the same location over and over again, you can store those intermediate values with the new `Cache()` parser:

```csharp
var parser = Cache(myParser);
```

Cached results are based on the reference identity of the parser and the location where the parse is performed. A complication with this is that when we retrieve a previously-cached result, we have to fast-forward the input stream to the point where the parse would have completed. Luckily we have cheap and fast `ISequenceCheckpoint` continuations for exactly this kind of purpose.

The cache lives in the `IParseState<T>` object, and is cleared after the parse is over unless you save the cache instance and manually reuse it in the next parse. 

I had originally envisioned a caching feature where you could turn it on globally and add packrat-style caching to all parsers automatically. That would have been nice, but would have added a lot of code complexity to every single parser in the system and hurt performance even when the feature was turned off. For now, I am accepting this solution where we can insert caching into the parser graph as needed with an explicit `Cache()` parser. In the future maybe I will find a more elegant way to properly integrate caching without having to create a separate parser instance for each location that needs it.

## Sequence Optimizations

Several of the `ISequence` types were not originally designed with large inputs in mind. If you've read my [previous post](http://whiteknight.github.io/2021/11/09/pocharstreams.html) you'll know that I have done a huge amount of work to optimize sequences. 

Stream-based sequences now use a single buffer and re-fill it in-place instead of maintaining a linked-list of all buffers. This is great for parsing large files or streams because memory consumption stays low. On the other hand there is a chance for pathological performance if you keep advancing and rewinding around the boundary of a buffer. Now when we rewind to a previous position, we actually "read behind" a few extra characters so small rewinds at the beginning of a buffer don't cause an immediate refill of the whole buffer. 

I have also removed caching of old values from all the other sequence types *except* `EnumerableSequence`. This sequence uses an `IEnumerable<T>` internally, and `IEnumerable` doesn't support rewinding to arbitrary positions once enumeration has started. I suppose *I could* dispose the old enumerator, and then do `.Skip(position)` to get back where we were, but for many types of enumerators that's a `O(n)` operation for every rewind! Right now `EnumerableSequence` is going to buffer all values, which means a very long enumeration will lead to large memory use. In a future 4.0 release that type will either be changed to operate on an `IList`/`IReadOnlyList` or it may be dropped entirely.

The net result of my changes is that streaming input, especially from large files, should be significantly faster in most cases.

## Other Optimizations

I have been optimizing the library throughout, to reduce object allocations and make various algorithmic improvements. This should be the fastest version of ParserObjects yet for most use cases.

## Coding Changes

During the ParserObjects 3.1 release cycle .NET 6 came out and C# 10.0 with it. I've already added a .NET 6 build target to the library, and have started making use of some C# 10.0 features. One of these is the `record struct` feature, which gives both brevity of declaration and the speed of a pass-by-value data type. I haven't yet made some of the bigger changes like file-based namespaces, but those are coming in v4.0. Already in my testing I've seen significant speedup most of the time running the .NET 6 build instead of the .NET 5 build or the .NET Framework 4.8 build. If you have the chance, try to use ParserObjects from inside a .NET 6 project so you get the fastest possible build.

## What's Next?

2022 is looming and I'm starting to think of [goals that I want to accomplish in the new year](https://en.wikipedia.org/wiki/New_Year%27s_resolution). ParserObjects is (currently) a big part of what I'm planning, though exactly what the short-term development plans are is still unclear. There are some features I want to add, and I also want to start making some breaking changes and cleanups throughout the library to account for lessons I've learned since v3.0 came out last year. I'll post more as my plans start to come into clearer focus.
