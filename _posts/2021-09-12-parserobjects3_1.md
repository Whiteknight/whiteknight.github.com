---
layout: post
categories: [projects]
title: ParserObjects 3.1 Development
---

[I mentioned a while ago](http://whiteknight.github.io/2021/01/09/earleyalgorithm.html) that I was starting to work on an implementation of the Earley algorithm for my [ParserObjects](https://whiteknight.github.io/ParserObjects/) project. Today I'm announcing development of the 3.1.0 release of ParserObjects, which includes the Earley parser and a lot more. This version has not been released yet, but I have gotten pretty far along in the development cycle and I wanted to talk about what to expect.

## Multi-Parsers and Multi-Results

The Earley implementation is the first of a category of what I'm calling "multi parsers" which can produce multiple possible results starting from the current input location. In the literature this is called a "parse forest". Earley can produce a parse forest by simultaneously keeping track of all possibilities as the parse progresses.

I have modified the Earley algorithm to play nicely with the ethos of my ParserObjects library: Parsers should be composable from smaller parts. This means that the Earley implementation needs to expect to be composed together with other parsers and cannot assume it commands the entirety of the remainder of input. If we have a precedence-less expression grammar like the following:

```
E := <number>
E := E '+' E
E := E '*' E
```

And we have the input sequence `4+5*6` the Early parser is going to return all of the following parse results:

1. `4`
2. `4 + 5`
3. `(4 + 5) * 6`
4. `4 + (5 * 6)`

At this point it is the responsibility of the next parser in line to try and figure out what to do. Do we select one "best" result and proceed with the parse from there? Or do we keep all results as potentially valid starting points and try to keep the multi-parse going forward?

ParserObjects defines two new abstractions. `IMultiResult` is similar to `IResult` except it includes multiple possible results from a parser including position information so the parse can be restarted from any of them. The `IMultiParser` is similar to the existing `IParser` but returns an `IMultiResult` instead of an `IResult`. Once you have an `IMultiResult` you can choose how to proceed from there. 

The `.ContinueWith()` method allows you to keep all possible results and continue the parse from each, extending your parse forest forward. As you attempt additional parse steps, some of these branches will fail and be pruned out of the forest. You can continue with an `IParser`, or even another `IMultiParser`, as your needs dictate. `.ContinueWith()` is analogous to `IEnumerable.Select()` (if you continue with an `IParser`) or `IEnumerable.SelectMany()` (if you continue with an `IMultiParser`).

If you want to choose one "best" result to continue with only, you can use some of the new extension methods like `.First()`, `.Longest()` or `.Select()` to pick just one result to move forward with. If you expect your parser to only have returned one result, you can use `.Single()` to access it.

Our expression grammar from above can be turned into a parser like this:

```csharp
var expression = Earley<int>(symbols =>
{
    var plus = Match('+').Named("plus");
    var star = Match('*').Named("star");

    var expr = symbols.New("Expr")
        .AddProduction(UnsignedInteger().Named("literal"), n => n);

    expr.AddProduction(expr, plus, expr, (l, _, r) => l + r);
    expr.AddProduction(expr, star, expr, (l, _, r) => l * r);

    return expr;
});
```

Now let's combine that with a parser which matches the name of a unit, and just take the first valid result:

```csharp
var unit = First(
    Match("feet"),
    Match("inches"),
    Match("meters")
);

var parser = expression.ContinueWith(unit).First();
```

Now if we have the input string `4+5*6inches` we will end up with a derivation that has `(54, "inches")` as the final result. We discard expressions which are too short and don't allow the `units` to match (`4` and `4+5`) and then take the first expression derivation that remains (`(4 + 5) * 6`) to continue the parse with. Or you could replace `.First()` with some other selection criteria to better control how your parse proceeds. There's a lot of power here.

As a general note, as a language designer it behooves you to make your grammar as unambiguous as possible. You should define rules in your grammar to ensure there is only one valid way to parse a given input sequence. However as a library designer, I cannot make the assumption that every grammar implemented with ParserObjects will be always be unambiguous or even that ambiguity is not desired in all cases. Part of the problem you are trying to solve might actually require exploring multiple ambiguous parses of a single input, especially when we start talking about natural language processing or dealing with legacy data file formats. The algorithm allows for ambiguities and I, as the library designer, must try to present them to you in a way that is straight-forward to deal with. The rest is up to you.

## Parse Statistics

I'm starting to introduce some basic functionality to get statistics and other metadata about a parse. Right now only the Earley parser implements statistics, and these values are not propagated from parser to parse. If you want to see them, you have to examine the `IMultiResult` directly coming out of the Earley parser:

```csharp
var statistics = result.TryGetData<IParseStatistics>().Value;
```

The values in this object will contain lots of information about how many objects are created and operations are attempted. Right now this is more of a curiosity and an aide for my own personal optimization efforts, but it might be expanded to be more useful in the future. 

Also notice that I made this `.TryGetData<>()` method as general as possible. In the future we might be able to attach other forms of metadata to a parse result as well. 

## End Sentinels

One of the design principles of input sequences, since the beginning of the library, is that parsing should be able to proceed without having to check at every single input position whether we've reached the end of input. I didn't want every single `c = input.GetNext()` call on an input sequence to be wrapped in a `try`/`catch` block or to be followed by `if (IsValid(c))` check. When input sequences reach the end of input, they set a flag `.IsAtEnd` which can be checked where necessary and they return a special end-sentinel value which can be treated like normal input but probably won't match any of your grammar rules. In fact all input sequences are effectively "infinite" in that you can continue to read from them past the end of input without issue, but you will just keep getting the end sentinel value over and over again.

In this way, handling of the end of input can be done in a more natural and simple way. There are parsers like the `End()` parser which match at the end position and fail everywhere else, `Any()` which does the opposite, and most other parsers would just reject the end-sentinel value as not being a match. You don't need to write special code all over your parser to detect unexpected end of input. Things will probably just work out the way you expect them to.

Selecting the end sentinel value to use seems straight-forward. You can just use `default` values for most types, like `null` for classes or `'\0'` for chars, etc. But there are times when these defaults don't make sense. Maybe you're reading a binary data file where `'\0'` is used as padding for as a field separator, or you have are reading a stream of tokens and expect a `new Token(TokenType.End)` not a `null` value.

As of the 3.1.0 release, all input sequence types in ParserObjects allow for custom end sentinel values. Here's an example of setting a custom character as the end sentinel:

```csharp
parser.Parse("test input", endSentinel: 'X');
```

## Sequence Checkpoint Improvements

`ISequence` reports both a number of `.Consumed` input items and a `.Location` for it's current position. As of 3.1.0 the `ISequenceCheckpoint` also includes these values so you can keep track of where those checkpoints point to. This is especially handy for `IMultiResult` when you have multiple possibilities floating around and want to know what the state is of each.

## Line-Ending Normalization

One thing that I've always hated about cross-platform software is that simple things like text file line-endings are different. You can't just read a text file on Linux the same as you can on Windows because one uses `'\n'` and the other uses `"\r\n"` for a newline. One requirement that I had with ParserObjects from very early on was that I wanted to abstract away line endings and normalize everything to always just use `'\n'`. Input sequences all look for `"\r\n"` (and also just `'\r'` in case you ever work with a really wonky old file) and replace them with just `'\n'`. For a lot of text-processing purposes this is great. You can ignore the details of line-endings and just focus on parsing the meaningful text of the file.

However there are a few situations where you might want to keep those pesky `'\r'` characters around, including utilities where you're trying to detect which line-endings are in use! With that in mind, ParserObjects now also includes an ability to turn off line-ending normalization:

```csharp
parser.Parse("test\r\ninput", normalizeLineEndings: false);
```

Again, I recommend you keep line ending normalization on for most use-cases. You don't want your parser to break unnecessarily on a new input file just because the person who wrote that file was using a different operating system. Unless `'\r'` and `'\n'` characters are specifically meaningful in your context, consider leaving line ending normalization turned on.

## Onward

The Earley implementation appears to be, through my own testing, complete and correct. However there are a few opportunities for optimization that I want to explore in the future. There are also several features that I had originally intended to have available in this release but ended up scrapping for various reasons. I had wanted a general-purpose memoization feature and I also wanted to provide an implementation of the GLL parsing algorithm. GLL, like Earley, is also a multi-parser so a lot of the groundwork is laid. I also thought that working on both simultaneously would help to flesh out my abstractions a little better. My initial implementation of GLL left a few questions to answer and presented a few design challenges, so I decided to push that back and focus on something I was more confident about completing. Memoization is likewise something that would be very useful to have, but finding a good and easy way to integrate it into existing parsers and being able to toggle it on or off as desired, created a few design challenges that I wasn't prepared resolve right now. These things will be the subject of later releases.

Expect a 3.1.1 when inevitable bugs are found. Expect 3.2.0 with some new features that didn't make it into this release. Expect 4.0.0 when I decide to delete everything in exasperation and fix all my mistakes in a single big rewrite. 
