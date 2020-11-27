---
layout: post
categories: [projects]
title: ParserObjects 3.0.0 Development
---

After the [2.1.0 release of ParserObjects](http://whiteknight.github.io/2020/11/10/parserobjects2_1.html) I thought I was going to put the thing on the shelf for a while and work on something else. But there are one or two ideas kicking around in my head that required breaking changes, so I created a branch for v3 and started hacking. And hacking. And hacking. Next thing I knew there were several big breaking changes and a whole slew of new features. Here I'm going to describe some of what is coming in v3.

**Important Note**: v3.0.0 is *completely backwards **incompatible*** with v2.x. If you are one of the dozen-or-so people using ParserObjects and you are on v2.x or earlier, the upgrade might not be a smooth one.

## Namespace Flattening

The change that precipitated the v3 development branch was my unhappiness with the way classes were organized. I hadn't put a lot of thought into the folder or namespace structure of the library, and I wanted to flatten things and make finding features simpler. So now this...

```csharp
using static ParserObjects.Parsers.ParserMethods<char>;
using static ParserObjects.Parsers.Specialty.LineParserMethods;
```

...becomes...

```csharp
using static ParserObjects.ParserMethods;
using static ParserObjects.ParserMethods<char>;
```

Not only have classes been moved to the root namespaces, but several classes containing static methods have been combined. The net result is that you should be able to use ParserObjects with far fewer and simpler `using` directives in your code.

## Sequence Checkpointing

In the v1.x and v2.x versions, the way we returned unused data to the input stream was through the `.PutBack()` method. If you read a value from the input and needed to un-read it, you could call `.PutBack()` to put the item back onto the stream. The problem comes when you read ahead many input tokens, reach a failure, and need to roll all the way back. In these cases we had to keep track of all input tokens in a stack, and then rewind by calling `input.PutBack(stack.Pop())` over and over again in a loop. Rewind operations were `O(N)`, which could be terrible for complex parsers. This all was done through the use of the window sequence decorator:

```csharp
var window = new WindowSequence<TInput>(input);
... attempt a parse, using window as the input sequence ...
window.Rewind();
```

Here `WindowSequence` is a decorator that wraps the source sequence and handles all that stack logic. This also adds a problem that if we have a Window, and recurse into another parser which requires a window, we will add another layer. We end up with a linked list of window calling window calling window, again leading to awful performance in some situations.

In the new v3.0.0 code, this mechanism has all been rewritten to use a checkpointing mechanism instead. When we call `.Checkpoint()` on a sequence, it takes a snapshot of the state of the sequence at that point. Then later when we call `checkpoint.Rewind()` it overwrites state directly without having to push characters around in loops.

```csharp
var checkpoint = input.Checkpoint();
... attempt a parse, using input as the input sequence ...
checkpoint.Rewind();
```

The one downside to this new implementation is that where `WindowSequence` was easy to reuse as a simple decorator, in the new version of the library every sequence must provide its own implementation of `.Checkpoint()` and there's almost no opportunity for code sharing. It's a small issue, but it just goes to show that there's no free lunch.

### Location Fixes and End-Of-Input

Related to the checkpointing work, I went back through all the different input sequence types and fixed handling of metadata including better tracking of `.CurrentLocation` and better detection of end-of-input. The `.IsAtEnd` property should be set to `true` immediately after the last item from the sequence was read, which meant most sequences needed to actually read one character ahead of the current position to know if the end of input was coming up. This readahead could be broken in some situations if we get to the end but then `.PutBack()` some characters. This is fixed now for all sequence types.

`.CurrentLocation` was a tougher nut to crack. The trickiest part is remembering which column was the last column of the previous line, in case the user called `.PutBack('\n')`. You can't just do `_line--` and call it a day, you have to actually remember what the column was at that point and roll back to it. Plus, I wanted this to happen in `O(1)` time without having to go back and count how many characters were in that line. To make this work most input sequences make use of a [Ring Buffer](https://en.wikipedia.org/wiki/Circular_buffer) or "circular buffer", a relatively uncommon data structure. Ring Buffers are fixed-size and use modular arithmetic, so when the buffer overflows, old data is automatically overwritten. In this case, the ring buffer keeps track of the last column number of the previous `N` lines, so that we can quickly jump back to previous values. If the user calls `.PutBack('\n')` too many times they will overflow the ring and get garbage data, of course. The ring buffer implementation was included in previous releases of ParserObjects, but I have finally put significant testing around it and fixed a few bugs.

## Contextual State

Another nice benefit from the sequence checkpoint rewrite is that I could refactor based on the knowledge that the input sequence object is the same object throughout the entire parse. It doesn't sometimes get wrapped in a decorator. The `Parser.Parse(ISequence<T> input)` method is replaced with `Parser.Parser(ParseState<T> state)`, and the input sequence is now found at `state.Input`. I can attach other data values besides just the input sequence to this state object. Specifically, I can add a slot for contextual data to be stored by the user. I have added and am evaluating mechanisms to interact with internal state, such as setting state at various points in the parse, and using saved values to effect the parse. I'll post more examples of this in future posts. 

## Sequential Parser

There are two big headaches with combinators:

1. It's a pain to debug them if something isn't going the right way and
2. They don't really play nicely with procedural logic.

The `Examine` parser was a quick attempt to address the debugging issue, you can add an examine call anywhere in your parse tree and see what the inputs and outputs are at that point. However, this isn't the best solution for the trickiest problems. Now in v3.0 we can use the new `Sequential` parser to inject procedural code into the parser graph. Here's an example lifted directly from the test suite:

```csharp
var parser = Sequential(s =>
{
    var first = s.Parse(Any());
    char second;
    if (first == 'a')
        second = s.Parse(Match('b'));
    else
        second = s.Parse(Match('y'));
    var third = s.Parse(Match('c'));

    return $"{first}{second}{third}".ToUpper();
});
```

This example will match string `"abc"` or something like `"xyc"`, and you can set a breakpoint anywhere in there that you want. A failure of any of the parsers jumps control back up to the Sequential parser, which will return failure, in keeping with the general flow control patterns of other parsers.

## Function Parser

Related to the `Sequential` Parser, though with a different focus, is the new `Function` parser. This parser takes a callback, which gets the raw input sequence and can perform any parsing logic desired. There's absolutely no structure built around it, so it won't automatically jump out on failure and doesn't provide helpers for working with `IParser`s. The Function parser is useful in places where you want to use your own parsing algorithm from scratch. For example, a stack-based algorithm to convert Reverse Polish Notation to a tree, or something like the [Shunting Yard Algorithm](https://en.wikipedia.org/wiki/Shunting-yard_algorithm) to parse mathematical expressions can be implemented using `Function`. Here is some code to implement an RPN calculator, pulled directly from the unit test suite:

```csharp
var parser = Function((t, success, fail) =>
{
    var startingLocation = t.Input.CurrentLocation;
    var stack = new Stack<int>();
    while (!t.Input.IsAtEnd)
    {
        var token = t.Input.GetNext();
        if (token == null)
            return fail();
        if (token.Type == RpnTokenType.Number)
        {
            stack.Push(int.Parse(token.Value));
            continue;
        }
        if (token.Type == RpnTokenType.Operator)
        {
            var b = stack.Pop();
            var a = stack.Pop();
            switch (token.Value)
            {
                case "+":
                    stack.Push(a + b);
                    break;
                case "-":
                    stack.Push(a - b);
                    break;
                case "*":
                    stack.Push(a * b);
                    break;
                case "/":
                    stack.Push(a / b);
                    break;
            }
            continue;
        }
    }
    return success(stack.Pop());
});
```

It's a simple and canonical example of an algorithm which now plays quite nicely with the rest of ParserObjects.

## Pratt Parser

It's been on my TODO list for a while, but I finally sat down and implemented a [Pratt parser](https://en.wikipedia.org/wiki/Operator-precedence_parser#Pratt_parsing) for ParserObjects. The Pratt parser is pretty versatile, but it will find the most immediate use in parsing mathematical expressions. The downside is that it can be a little bit tricky to use correctly, you have to keep track of precidence and associativity values for all rules yourself. It's still early in the development cycle, but the following code snippet to parenthesize equations, among several others, is currently working in the test suite:

```csharp
var number = Digit().Transform(c => c.ToString());

var target = Pratt<char, string>(number, config => config
    .AddInfixOperator(Match('+'), 1, 2, (l, op, r) => $"({l}{op}{r})")
    .AddInfixOperator(Match('-'), 1, 2, (l, op, r) => $"({l}{op}{r})")
    .AddInfixOperator(Match('*'), 3, 4, (l, op, r) => $"({l}{op}{r})")
    .AddInfixOperator(Match('/'), 3, 4, (l, op, r) => $"({l}{op}{r})")
);
var result = target.Parse("1+2*3+4/5");
result.Value.Should().Be("((1+(2*3))+(4/5))");
```

The Pratt implementation I have put together is pretty focused on use with mathematical expressions, though it can be a little bit more general than that. The Pratt algorithm is so simple that if you need more flexibility you can probably put a custom version together. Feel free to crib the source code from the repo and make your own modifications.

## List and SeparatedList Parser Simplification

The `SeparatedList<TInput, TSeparator, TOutput>` parser has been simplified, because it turns out we don't care about the output type of the separator parser. All we really care about is whether the separator exists or not. Now you can rewrite that as `SeparatedList<TInput, TOutput>` and ignore the separator.

Both `List` and `SeparatedList` parsers now return `IReadOnlyList<TOutput>` instead of `IEnumerable<TOutput>`. While `IEnumerable` is more general, the list type allows us to reference results by index without calling `.ToList()` first, which is more powerful in many ways and saves an allocation.

These changes are relatively small but, considering we are already making breaking changes, I thought this was a good time to make them.

## Flatten Parser

One of the features from the v2.x releases which will be missing in v3.0 is the `FlattenParser`. This parser was problematic for a bunch of reasons: It was the only parser which had mutable state which meant recursion into it would have caused serious problems, and it was one of the only parsers which wrapped another parser and would succeed or fail in part depending on the *returned value* of the inner parser. The flatten parser would fail if the inner parser returned an empty collection, which is a very common occurance (think about the javascript `[]` empty list construct, for example). I have removed this parser, there are better ways to acheive the same result without creating the same problems.

## Error-Handling and Logging

I'm still in the middle of it, but I'm adding a logging feature to the new `ParseState<T>`, so that parsers can log current state and things that are happening in the middle of the parse, to help with debugging. 

`IResult<T>` (previously `IParseResult<T>`) now includes much more metadata, especially when a parse fails: a reference to the parser which produced the result, an error message, and a detailed `.ToString()` override. 

Along with the `IResult<T>` improvements, there is now a `TransformError` parser, which can be used to adjust an error result. You can, for example, enrich the error message and metadata to add better diagnostics, or you can transform error results into success results. For example, you can turn an error result into a success result, with a synthetic result value containing diagnostics.

## Misc Changes

`PositiveLookahead` parser now returns the value it parsed (even if the input is not consumed). This can be valuable in some situations where we want to use that value to select the next behavior, or save that value into contextual state, etc. Notice that these two calls are now functionally identical (though the first one is more efficient):

```csharp
var p = parser1.Choose(r => parser2);
var p = PositiveLookahead(parser1).Chain(r => parser2);
```

The new `Combine` parser works a lot like `Rule` but returns the raw list of result objects. This includes extension methods for various `ValueTuple` types, so you can write something like this:

```csharp
var p = (p1, p2, p3, p4).Combine().Transform(list => list[2]);
```

I have done a lot of work throughout to improve usability and readability of parsers, and to make better use of generic type inferencing to simplify parser declarations. 

I have also been doing a lot of work on the documentation site, which will be available as soon as the v3 branch merges to master.

## Whatever

As soon as I decided to break backwards compatibility, I opened the door to making a whole lot of changes to improve the situation. The old saying goes "no plan survives contact with the enemy", and having used ParserObjects downstream a lot more since 2.0.0, I have found that some of the things I had planned didn't work the way I wanted them to.

Some of the new features I added, such as the `Pratt`, `Sequential` and `Function` parsers could have been added in the 2.x branch, but considering the effort to move things around after the flattening and reorganization, I decided to just implement them in v3 instead. 

There are a few more little features on my list to add, and lots of spit and polish to make sure all the new features I have are clean, well-tested, and easy to use. As always, the process of going from "feature complete" to "ready for release" is longer and harder than just writing all the new code in the first place, so don't expect 3.0.0 to hit nuget with any haste.

