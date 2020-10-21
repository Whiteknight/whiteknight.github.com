---
layout: post
categories: [projects]
title: ParserObjects v2.0.0
---

When I'm preparing a release, especially one with a major version bump, I like to go through a few steps: Go through and cleanup the code, resolve or remove TODO notes, make sure code coverage is up to a reasonable level if I've been slacking on that front, and make sure there's plenty of user-facing documentation so downstream users can use the library effectively. Going from "feature complete" to "ready to release" is almost more work than everything else before it. But, I think, it's effort well spent.

After using ParserObjects v1.0.0 in the wild for a couple months, I had a few things I wanted to change. I had originally thought about a minor-version bump instead, but some of the changes I wanted to make were breaking so I started on the long path to v2.0.0. I released that a few weeks ago (ParserObjects v2.0.0 actually released before StoneFruit v1.0.0, and CastIron v2.0.0-alpha3 from my previous two blog posts, but I did the blog posts in the wrong order and we all just have to deal with that).

Here I'm going to list a couple of the things that have changed in the library.

## ExamineParser

A particularly frustrating issue was in debugging parsers, when something was failing it was hard to figure out exactly where the problems were happening. To this end, I added the `ExamineParser` which allows inserting arbitrary callbacks into the parser graph.

```csharp
myParser
    .Examine(
        s => { ... called before the parse ... },
        s => { ... called after the parse ... }
    );
```

It's a handy little tool to keep around for spot-checking the parse and figuring out what the state is at each point and why something isn't working the way you expect.

## ParserMethods Extensions

A big breaking change that actually precipitated the 2.0.0 release tract was renaming the `ParserMethods` class to `ParserMethods<TInput>`. This allows us to cleanup the parser declaration significantly, because you can omit the input type on all the parser extension methods. So instead of this:

```csharp
using static ParserObjects.Parsers.ParserMethods;

var match = Match<char>("test");
```

We can now write this:

```csharp
using static ParserObjects.Parsers.ParserMethods<char>;

var match = Match("test");
```

It's not a huge improvement, but it does make the code cleaner and I'm happier with it like this. Parameterized static usings like this aren't a commonly-used feature in the language, but it's a nice little tool to keep in your back pocket to help with code cleanup like this.

## BNF Stringification

By implementing a visitor over the object graph, ParserObjects can produce a bit of almost-but-not-quite pseudo-BNF output to represent your grammar. While it isn't perfect, it is generally good enough to help double-check your work and make sure you are parsing what you think you are parsing. While I had this at the 1.0.0 release, I have made a few significant improvements to the output which will produce a much nicer output.

```csharp
var bnfString = myParser.ToBnf();
```

## Going Forward

This wasn't a huge release in terms of features or changes, just a lot of little cleanups and fixes. One thing that I did improve significantly is my local tooling and process. On Github I enabled [Dependabot](https://github.com/dependabot) which was a very simple process and I recommend it for anybody whose projects have external dependencies. Locally, I've added [OpenCover](https://github.com/OpenCover/opencover) which is a code coverage tool, and I'm feeding the results of that into [SonarQube](https://www.sonarqube.org/) to get reports and code-improvement recommendations. I'm running SonarQube Community edition locally via Docker, so I can bring it up when I need it and down when I don't. I used to use FxCop a long time ago but gave that up because I was unhappy with it. SonarQube is much nicer, and I'm glad to have a code-quality tool added back to my local dev toolbox again.

As far as the future goes, I've been kicking around a few ideas for things to work on next: A tool to create a parser graph automatically from a BNF string or even a Regex, and maybe something like a built-in operator precidence parser or Pratt parser to help simplify the work of parsing math formulas with all the precidence and associativity rules. Right now though, 2.0.0 is out and it's stable, and there are other projects I want to focus more attention on in the coming months.
