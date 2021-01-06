---
layout: post
categories: [projects]
title: ParserObjects 3.0.0 Released
---

The perfect is the enemy of the good. When I'm working on my own personal projects, in my own spare time, it's easy to completely discard the trappings of professional work: deadlines and budgets. It's easy to keep finding little things to keep poking at, finding all sorts of little excuses to avoid cutting the release today because everything isn't exactly perfect. It's not just that I am such a perfectionist that I would prevent myself from releasing anything, but also that my concept of what "perfect" is changes over time. I have a feature that I tweak until I think it's acceptable, but then I decide that the design is all wrong so I start all over again. The [3.0.0 development cycle for ParserObjects](http://whiteknight.github.io/2020/11/27/parserobjects3dev.html) has also freed me to make a bunch of breaking changes, which means there are even fewer constraints. I can just keep changing things and not releasing anything forever if I want to!

Part of me does want that.

On the other hand, I do want to get this thing closed down, so that I can move on to other projects. So, today, I'm announcing the 3.0.0 release of ParserObjects.

## What's In It

The [previous blog post](http://whiteknight.github.io/2020/11/27/parserobjects3dev.html) covers a lot of what's new in this release, though many things have changed since then as well. Here's a brief recap of what I mentioned last time:

* **Namespace Flattening**: Folders and namespaces are reorganized with an eye towards simplicity and ease.
* **Sequence Checkpointing**: Memento/Continuation implementation for jumping around in sequences in `O(1)` time.
* **Contextual State Data**: You can set and retrieve contextual data during the parse.
* **Sequential Parser**: You can compose parsers using procedural code, and set breakpoints in the middle.
* **Function Parser**: You can implement any parse algorithm you want, without form or restriction.
* **Pratt Parser**: You can use the powerful Pratt operator-precedence parsing algorithm.
* **Error Handling and Logging**: You can log messages during the parse to follow what is going on.

This already is a pretty good list of features and changes, but I've done even more since then.

### C# 9.0

The project is updated to dual-build .NET Standard 2.0 and .NET 5.0, and started making use of C# 9 features internally. There's more to do in this regard, especially a better use of `record` types throughout to help enforce immutability, but so far I have been quite happy with the new language additions. The reason for targeting both .NET Standard 2.0 and .NET 5.0 is that I want the former for maximum portability and I want the later so that I get "native" implementations of C# 9.0 features and better performance. If you're using the .NET Standard 2.0 build, some of those features are implemented with polyfills. Both runtimes are built and tested separately to make sure everything works everywhere (The test suite for .NET 5.0 runs about 50% faster than the same test suite for .NET Standard 2.0, so that's an interesting data point to keep in mind).

ParserObjects doesn't have any external dependencies besides the .NET Runtime itself, so there shouldn't be any need to abandon .NET Standard 2.0. There is the small detail that all the .NET Framework versions which target .NET Standard 2.0 are running out of support lifetime, but that's an issue for downstream consumers to deal with.

### Peek Parser

The new `Peek` parser is almost identical to the `Any` parser, except it only peeks at the next input value but does not consume it. This parser is used mostly for internal composition purposes, but can find some use in your downstream code as well.

### Predict Parser

There's now a `Predict` parser, which peeks at one value of lookahead and selects which parser to invoke in response to that. `Predict` is implemented on top of `Chain` and `Peek` internally, but gives a very friendly syntax. Here's some example code lifted directly out of the JSON parsing example in the test suite:

```csharp
valueInner = Predict<IJsonValue>(c => c
    .When(t => t.Type == JsonTokenType.OpenCurlyBracket, jsonObject)
    .When(t => t.Type == JsonTokenType.OpenSquareBracket, jsonArray)
    .When(t => t.Type == JsonTokenType.String, str)
    .When(t => t.Type == JsonTokenType.Number, number)
    .When(_ => true, Fail<IJsonValue>("Unexpected token type"))
);
```

Predict is useful when `First` might be too expensive to try various approaches, and a single value of lookahead can determine where we need to go next. It's also useful because `PREDICT()` is a common basic operation in academic parser literature, so this helps to make some algorithms more translatable from academic description to actual C# code. 

There is also a new `ChainWith` parser, which is similar to the `Predict` parser except you can specify whatever you want the leading value to be, not just a peek at the next input:

```csharp
var parser = Predict(...);
var parser = ChainWith(Peek(), ...);
```

`ChainWith` is strictly more powerful than `Predict`, but also a little more verbose.

### None Parser

The `None` parser wraps an inner parser, and rewinds the input sequence so that it consumes no input. This is useful in some situations where you want to test an approach before deciding what to do, especially when talking about something like `ChainWith`. As an example, `Peek()` is equivalent to `Any().None()` but shorter to type and faster to execute.

### Consumed Inputs

All `ISequence`s now have the ability to report the total number of consumed input values, and all `IResult`s now report how many input values were consumed to construct the current value. All parsers which loop or recurse are updated to inspect this value so that infinite loops can be avoided. 

### PutBack is Removed

`ISequence.PutBack` method is removed. This method is superseded by `.Peek()` or `.Checkpoint()`/`.Rewind()` in all cases. PutBack made the implementations of several sequence types significantly more complicated, and removed the ability to optimize `.Rewind` operations. With all rewind operations being optimized to be `O(1)` time complexity, many more operations present and future can be rewritten to make heavy use of rewind. Regular Expressions and Trie operations use rewind now, instead of looping over `.PutBack` to return unused values to the sequence. 

### Try Parser

While ParserObjects generally tries to use Exceptions sparingly, there are many occasions where parsers provide callback delegates to the users, which can inject arbitrary code throughout the parser graph. Each of these may be a source of exceptions. I have added the new `Try` parser to be able to catch exceptions from user callbacks, properly rewind the input sequence, and control how the exception is handled. Exceptions can either be re-thrown to bubble up to another handler, or they can be turned into failure results so the parse attempt can continue.

### Misc Changes

* `LeftApply` and `RightApply` are both improved, optimized, and they take quantifiers to control arity.
* `IParser.ReplaceChild` method is removed from all parsers. Now replacements are made using the `ReplaceableParser` only.
* Several bugs in `Regex` were found, fixed, and tested
* Several other bugs and inconsistencies are fixed, unit test coverage is expanded significantly, and code quality throughout has been improved.

## Updating

Updating from 2.x versions of the library to 3.0.0 will require some code changes, mostly replacing `using` directives to point to the new namespaces. Some additional changes will be required around `Produce` and `Empty` parsers and a few others. For the most part it's not a difficult process, though it is also not completely free. As I mentioned in the previous post:

```csharp
using static ParserObjects.Parsers.ParserMethods<char>;
using static ParserObjects.Parsers.Specialty.LineParserMethods;
```

...becomes...

```csharp
using static ParserObjects.ParserMethods;
using static ParserObjects.ParserMethods<char>;
```

## Going Forward

I'm pretty happy with the state of this library right now, and I'm hoping that ParserObjects v3.0.0 can start to attract some attention and some real-world use. I'm not planning to sit and rest here, however. I'm already working on a few new features for this library which, if successful, could form the basis of a 3.1.0 release. Among some of the big features I'm planning are support for memoization and possible support for parsers which return multiple alternative values.

There are two algorithms I'm looking at in particular which may return multiple values: [Earley](https://en.wikipedia.org/wiki/Earley_parser) and [GLL](http://www.cs.rhul.ac.uk/research/languages/csle/GLLparsers.html). Both of these things would bring quite a bit of power to the ParserObjects library, both would require significant adaptation to work, and both of which were mentioned in [my 2021 resolutions](http://whiteknight.github.io/2021/01/01/2021resolutions.html) as things I wanted to learn and implement. Maybe things work out and you'll see one or both implemented in ParserObjects v3.1.0. Maybe it doesn't work out and I do something else with them.

I also have some things planned for the downstream consumers of this library, I'll post more about those as I start to make progress on them.
