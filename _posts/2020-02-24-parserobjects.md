---
layout: post
categories: [projects]
title: ParserObjects
---

Let's talk about parsing. I've been doing a lot of it lately. I've already talked about my [SQL Parser](/2019/11/09/sqlparser.html) project and the little [CSPath](/2019/11/11/cspath.html) project I threw together also. Then there were two other projects that I've been working on but haven't written blog posts about, yet. At some point I realized that in all these projects I was either rebuilding or outright copying the same classes over and over again. If that's not the impetus to create a new library, I don't know what is.

So I got to work and now the [ParserObjects](https://github.com/Whiteknight/ParserObjects) library is barreling towards a 1.0 release pretty quickly. It contains a bunch of ideas that I've seen in other projects and a few things that I've been building, rebuilding and refining over the past year. In short, it's a library of *streaming monadic parser combinators*, and I'm pretty happy with it so far.

## The General Idea

The Recursive Descent technique is both simple and powerful. If you understand some basics of grammars and parsing, and you can make a function, you can probably do Recursive Descent. Basically every rule in your grammar becomes a method, and you can call one rule from another by calling the correct method. It's old-school procedural, but it works.

What if we wanted to do the same basic thing as Recursive Descent, but using an Object-Oriented approach? That's called **combinators**. You start with a small parser which does just one thing, like recognizing an operator or keyword, and you combine those together in other objects. The object graphs you create eventually can encompass complex grammars. The best part of combinators, in my mind, is that with a little bit of preparation the code you write to build these object graphs look a heck of a lot like a BNF grammar. In other words, the description of the grammar and it's implementation are basically done in the same place at the same time.

(Compare to, for example, LALR parser generators where you write a grammar in one file, which goes through a translation step to become code to implement that grammar, but the grammar you write and the generated code are so different from each other in every way that it would be almost impossible to figure out which part of one corresponds to which part of the other.)

Add in some nice syntax from C#'s extension methods and `using static` declarations, and you can make a very nice parser indeed.

## Quick Example

A method in C# consists of a few parts. Here's an example:

```csharp
public int MethodName(MyArg arg)
{
    Statement1();
    Statement2();
}
```

A subsection of BNF for a C# method (minus several details) looks something like this:

```
statementList := <statement>*
normalMethodBody := '{' <statementList> '}'
parameterList := '(' <parameters> ')'
method := <accessModifier> <returnType> <Identifier> <parameterList> <normalMethodBody>
```

Translating that into Recursive Descent would require 4 methods plus methods for all the other rules we reference. But using combinators, we can define the parser like this:

```csharp
var statementList = Statements
    .List();

_normalMethodBody = Rule(
    Operator("{"),
    statementList,
    _requiredCloseBracket,

    produceNormalMethodBody
);

var methods = Rule(
    Attributes,
    _accessModifiers,
    returnTypes,
    _identifiers,
    _parameterLists,
    methodBody,

    produceMethod
);
```

If you put everything on one line and squint a little, it really does look a lot like the BNF. Every rule becomes a variable instead of a method, and you can string them together quite easily in argument lists. Then you pass your parser outputs to some function to produce the output result and you're done.

## Working with Object Graphs

Besides the nice readability of combinator-based parser declarations, there are other benefits as well. Since your parser is an object graph, you can work with it using a variety of tools and techniques. You can traverse and inspect the graph with the Visitor pattern, or you can use graph-editing operations to modify a grammar in-place. Imagine being able to modify your grammar to enable or disable experimental new features, or quickly adapt to different dialects without having to copy your whole grammar or fill it with a bunch of ugly conditionals! Or imagine being able to build up your parser object graph at runtime in response to user input or some kind of imported grammar string?

Need to read a CSV file where the columns are in a different order each time? Read the headers, map them to a list of parser objects, and insert them into your grammar. Need to work with request objects which are in a different dialect from different sources? You can modify your grammar to suit the particular needs of each one.

A lot of the work that I have been doing has been on these features of graph traversal and graph editing. I'm pretty happy with the results so far and I've been able to use them to good effect in a variety of projects already.

## Status

I'm trying to get the library to a basic, minimum-viable 1.0 release as soon as possible. I'm in the process of wrapping up a few loose ends, fixing old TODO notes, adding some documentation, and filling in the gaps in my unit test coverage. I am also trying to put together a bunch of example code for common parsing tasks like expression parsing (with multiple levels of precedence and different associativity) and data format parsing. I'm also putting in pre-made parsers for common tasks like "C-style identifiers" or "JavaScript-style numbers" and various dialects of comments, strings, etc.

Expect a 1.0 release in the next few days along with more blog posts of other projects I'm working on which build on top of ParserObjects to do some cool things.