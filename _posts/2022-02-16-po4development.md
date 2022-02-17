---
layout: post
categories: [projects]
title: ParserObjects 4.0 Development
---
I had been wavering back and forth about where to go with ParserObjects after the 3.1.0 release came out. Should I stick to adding some new features in a minor 3.2.0 release next, or should I make the breaking changes I've been avoiding and go for a major 4.0.0 release. I silently made this decision a while ago, merging the `v4` branch into `master`. Today I have pushed the first early alpha version to [Nuget](https://www.nuget.org/packages/ParserObjects/4.0.0-alpha1). I just didn't have enough new features that I could implement without making breaking changes to support them. Another big factor was that I had intended a GLL implementation to be the centerpiece of a 3.2.0 release, but I wasn't happy enough with how that was shaping up.

## On GLL

The thing with the GLL implementation is that I want ParserObjects to be *very easy to use* for a variety of tasks. I want downstream users to easily be able to create parsers either by composing existing parsers or by implementing the `IParser` interfaces in their own classes. Any time you're expecting people to create their own implementations of interfaces, you want to make sure those interfaces are simple and straight-forward. Ideally there should be relatively little boilerplate, no code duplication, and very straight-forward responsibilities. I'll be writing up another post about the topic soon, but suffice to say that I wasn't really able to meet my objectives with a GLL implementation at this time.

I am thinking about creating a separate nuget package to add GLL as an extension to ParserObjects. That way I can play with an implementation and interfaces separately from the main development cycle of ParserObjects. Again, I'll post more about this later.

## Parser Immutability

A big goal for the 4.0.0 cycle was to make almost all parser types immutable. Previously parsers all had to have a mutable `Name` property so that we could name parsers (for use in find/replace operations, BNF stringification, and a few other niche features). With mutable parsers something like `Any()` cannot cache instances because even though the parser itself has no real state and has the same behavior across all instances a user might rename the single cached instance, or might attempt to rename it multiple times silently smashing Name data that might be important. The old interface looked like this:

```csharp
public interface INamed
{
    string Name { get; set;}
}
```

But now the new interface is this:

```csharp
public interface INamed
{
    string Name { get; }
    INamed SetName(string name);
}
```

The new `SetName` method returns a new object with the given name and all other state the same. In other words, `SetName` is *Copy on Write* and we can safely cache instances of common parsers. Most parser types tend to be pretty small anyway, so this copy is not a problem if you aren't renaming parsers over and over again. We made this happen by re-implementing most parsers with the new `record` type from C# 9 and using the new `with` operator to copy the record but with only the Name changed:

```
public record MyParser(..., string Name = "")
{
    public INamed SetName(string name) => this with { Name = name };
}
```

`record` also brings several other benefits including auto-generated quality members, deconstructors, etc.

I said "make almost all parser types immutable" because there's one parser which is explicitly mutable: `Replaceable`. This parser explicitly allows modifications to the parser graph and so is not immutable. 

## Parser Ids

Every Parser instance now has a unique ID value. Before this, if you didn't set a unique `Name` on your parser, it would be very hard to find and uniquely identify it in the parser graph while you were debugging. Having a unique ID simplifies caching a lot as well.

## Function Parser

The `Function` parser is a very functional-programming kind of implementation, and many parser types feel more "natural" when implemented as a function instead of as a class or as a composition. In 4.0.0 I have actually re-implemented several parsers in terms of the `Function` parser. This approach uses less memory and simplifies several implementations in a significant way.

## Statistics and Custom Data

I have added several abilities to get data about the parse and individual results. For instance, you can get statistics about the input sequence including how many input items you have read, how many times you've backtracked, etc. I had been keeping these statistics internally for a while now, but I couldn't expose them until now because it would be a breaking change on the interface:

```csharp
var stats = state.Input.GetStatistics();
```

Likewise you can get information from the cache about cache hits and misses:

```csharp
var stats = state.Cache.GetStatistics();
```

Finally `IResult` and `IMultiResult` now have an ability to attach arbitrary data that might be handy for analysis or debugging, and this data can be accessed with the new `.TryGetData<T>()` method. This data is keyed by type, and is not propagated from one result to the next, so you really need to know exactly what you are looking for and where you are looking for it. But in some cases you can get really interesting details. The `Try` parser includes the `Exception` object for example, and the `Earley` parser includes some detailed statistics about the parser and production.

```csharp
var result = Try(...).Parse("...");
var ex = result.TryGetData<Exception>();
```

```csharp
var result = Earley(...).Parse("...");
var stats = result.TryGetData<Earley.IParseStatistics>();
```

## ListSequence

Previously there was an `EnumerableSequence` type that adapted `IEnumerable<T>` into `ISequence<T>`. The problem here is that `ISequence<T>` supports random seek while `IEnumerable<T>` doesn't. This means that we had to buffer values in a list, which potentially ate up a large amount of memory (if the underlying enumerable already was a list or array, we were double-buffering and doubling memory requirements). In 4.0 I have removed `EnumerableSequence` and replaced it with `ListSequence` which specifically requires and `IReadOnlyList<T>` and does not double-buffer.

## Misc Changes

I have removed every single use of the `else` keyword from the codebase. Part of the purpose of this library is to really show my code style and beliefs about how to structure code, and not having `else` is part of that. I had about a dozen uses of the keyword in the v3.1.0 release if I remember correctly, but now we are down to 0.

I have changed all files to use file-level namespaces, the new feature from C# 9. This helps remove a lot of indenting and per-file boilerplate.

I have renamed a few things that I wasn't happy with the names of. `ValueTuple.Produce()` extension methods have been renamed to `.Rule()` for consistency. `.ProduceRight()` and `.ProduceLeft()` methods in Pratt have been renamed to `.Bind()` and `.BindLeft()` respectively. 

There is a multi-parser version of the `Trie` parser now which returns all possible matches. Previously it only returned the longest match.

## Going Forward

I'm still digging through the codebase, looking for things to clean and improve. At the time of this writing Sonarqube appears to not yet support C# 9.0, so it's really unhappy about some of the changes I made. I would like to be able to run that analyzer before I cut a real release, and I want to work a bit more on test coverage anyway. I'm hoping to put 4.0.0 out in the next month or two.

After that? Who knows. I'm looking for things to work on. 