---
layout: post
categories: [projects]
title: ParserObjects 2.1.0
---

Today I released version 2.1.0 of [ParserObjects](https://github.com/Whiteknight/ParserObjects) to [nuget](https://www.nuget.org/packages/ParserObjects/2.1.0). This release added a few cool new features and sets the stage for bigger work in the future. 

## Regular Expressions

One of the biggest and most exciting features added to ParserObjects is **Regex support**. Now you can do something like this:

```csharp
var parser = Regex("a(b|c)*de");
var result = parser.Parse("abcde");
```

A lot of parsing tasks can be dramatically simplified if you can just match a string using a regex instead of parser combinators or recursive descent, or any other technique. I hadn't added regex support to ParserObjects before now for two reasons:

1. I couldn't just use the built-in `Regex` class to do it, because the parsers are operating on streaming input data not constant `string` buffers and
2. I didn't know how.

Item #1 meant that I needed to make a custom regex engine. I couldn't re-use the existing `Regex` functionality of the standard library. Plus, there are plenty of details of a regex that don't make sense in a streaming context. For example the regex parser is always expected to match from the current position in the stream and not search for a match somewhere down the line. The `^` anchor isn't really useful anymore because we're always anchored to the current location in the stream. It also doesn't make sense to do captures, because we are only dealing with complete matches, `( )` is just used for grouping and not capturing, and `(?: )` for non-capturing groups is redundant.

Combinators like those used in ParserObjects are very good at outputting trees. In my first attempt at implementing a regex engine, I parsed the pattern string into a tree, and then tried to traverse the tree in the regex matching engine to pull characters and match them. Things were a little shakey, but I was making progress until I had to implement backtracking. To implement backtracking, I would have needed my visitor to be able to jump back to arbitrary locations in the pattern tree, but also be able to put back characters it had matched along the way. It quickly became apparent that, while the approach might have been technically possible, it was definitely not going to be pretty. So, I decided to look up some prior art to find a better algorithm, when one landed at my feet. A [video explaining how to create a simple regex engine](https://www.youtube.com/watch?v=u01jb8YN2Lw) showed up in my youtube recommendations the very next day. For anybody who hasn't made a regex engine and would like to know how they work, I strongly suggest giving that video a watch and clicking some likes or subscribes or whatever.

It's obvious in hindsight: regexes aren't branching and chaotic like a tree, they're linear like a stack. I took the tree output of my pattern parser, traversed that to create a queue of state objects, and then was able to use that to create a simple but robust parsing engine. 

The code already supports all the normal quantifications (`*`, `?`, `+`, and `{3,4}`), some basic built-in character classes (`\w\W`, `\s\S`, `\d\D`, etc), grouping with `( )` alternations with `|` and the end anchor with `$`. I'll be looking into adding additional features in a future release.

## Character Class Parsers

I've added a few new character class parsers, to compliment some of the ones that were already available. Among these are `Letter()` and `Word()` to match exactly one, or several letter characters, respectively. There are also `UpperCase()` and `LowerCase()` methods and also `Symbol()` for all symbols and punctuation characters. These are all pretty straight-forward wrappers around methods in the `char` class.

## Choose and Chain

Two new features I added, also on inspiration from the [Low Level JavaScript YouTube channel](https://www.youtube.com/channel/UC56l7uZA209tlPTVOJiJ8Tw) are the **Chain** and **Choose** parsers. Chain parses an initial parser, and uses the output of that to select which parser to return next. Choose is almost identical, except the initial parser is matched, so input is not consumed. Both of these are interesting because they give the option to make a dynamic decision about what to parse next depending on what you have. A lot of this we could already do, in a messier way, with the `First` parser or the `LeftApply` parser, but Chain and Choose make the logic clearer and more readable in some cases. Here's an example showing `Chain` in action:

```csharp
var parser = Regex("number|name").Chain(type => {
    if (type == "number")
        return Combine(Match(':'), Digits()).Transform(v => v[1]);
    if (type == "name")
        return Combine(Match(':'), Letters()).Transform(v => v[1]);
    return null;
});
```

The alternative using `First` would be something like this:

```csharp
var parser = First(
    Rule(
        Match("number:"),
        Digits(),
        (_, v) => v)
    ),
    Rule(
        Match("name:"),
        Letters(),
        (_, v) => v
    )
);
```

It's arguable which approach is clearer and more readable between these two in this case, but as the complexity of the decision gets larger, I think Chain and Choose can be very helpful indeed.

One problem with these two parsers is that they don't play nice with some of the visitors, because the produced parsers from the callback method cannot be exposed to the visitors. So, things like find/replace and BNF stringification won't work with them.

## Function Parser

You can make parsers using a more functional style with the **Function** parser. This is basically just a wrapper to turn a function delegate into a parser object:

```csharp
var parser = Function(t => {
    var next = t.GetNext();
    if (next == 'a')
        return new SuccessResult<string>("AAA");
    return new FailResult<string>();
});
```

This is a great way to do work which can't be neatly mapped to combinators.

## Work Ahead

v2.1.0 is a nice little point release and I'm happy with the features that were added. Right now, my backlog of issues is starting to fill up with large and breaking changes, which means a v3.0.0 is probably coming down the road at some point. I would like to do a major reorganization of the code and make several changes throughout to improve usability, though I'm only planning to pick at these items here and there as I have time. Right now there just isn't a big sense of urgency, so I'll stick with what I have here and start working on something else.
