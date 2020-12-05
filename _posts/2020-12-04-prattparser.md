---
layout: post
categories: [projects]
title: Pratt Parser Implementation Notes
---

In [my previous post](/2020/11/27/parserobjects3dev.html) I mentioned  that I was developing an implementation of the Pratt parser algorithm for ParserObjects. Since that post I deleted what I had and completely rewrote the Pratt parser and (I hope!) dramatically improved both the power and usability of it. Today I'd like to talk a little bit more about this effort, how I got here, and what I've been doing.

## A Brief History

I was aware of the Pratt algorithm back in the Parrot days, when it seems like every single project had a parser component to it. I had read [Vaughn Pratt's paper](https://tdop.github.io/) on the topic, but I don't think I ever implemented one of my own (at least not for anything larger than a toy). I think some of the Parrot projects used Pratt parsers, or parsers similar enough that I couldn't tell the difference, but that was just code I was reading, nothing I wrote myself.

Since the beginning of the ParserObjects library, I had "Pratt Parser" on my TODO list. It was something that I knew would be able to help with some situations, especially operator-precidence, but I didn't know how much implementation effort it would take to make it happen. During the 3.0 development cycle, I stumbled across [this article on Pratt parsers](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html) which was enough motivation to try my hand at my own implementation. The example code there was written in Rust, which I actually don't see very much of and was happy to read. With a few little tweaks to handle the streaming `ISequence` input and some of the `IParser` invariants, I had a working sample. It was shortly after this that the previous blog post was published with the short code example:

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

The problem I had with this approach, while easy enough to implement and get working quickly, was it's rigidity. It could only really handle operators on numerical sequences, and those operators were sorted into only a few buckets for infix, prefix, postfix, circumfix and postcircumfix. There were several constructs which I couldn't add in a clean way. Something like the `? :` ternary operator, for example, is different enough that it doesn't fit cleanly into one of the aforementioned buckets, but not part of a larger family of operators that would warrant creating a whole `.AddTernaryOperator()` method to handle it. There are no other common use-cases I can think of that would use the same structure and might actually come up in common practice. I also wanted to handle a so-called "listfix" case, where something like a function parameter list contains values separated by commas, but which would be consumed in a list and combined together into a single production, not like infix where we would produce two items at a time. It also occured to me that this implementation, as written, couldn't really cover any other structures besides mathematical operators, at least not without a lot of additional special-cases added here and there.

I was prepared to just accept these limitations, put in a bunch of notes about limitations in the documentation, and gone ahead with the 3.0.0 release like that. However, I stumbled onto another blog post which showed a different, and more flexible version of the algorithm. [This blog post](https://eli.thegreenplace.net/2010/01/02/top-down-operator-precedence-parsing), with example code written in Python this time, showed a better way to structure the core parse loop, and break parse rules out into separate *parselet* classes to hold arbitrary parsing logic. A downside is that this new implementation wasn't written in a way that played nicely with ParserObjects. At least, not without some modifications.

The `IParser` implementations in ParserObjects all share a few basic invariants:

1. When the parser fails, it resets the input sequence to the state it was at before the parse was attempted. A parser trying and failing should leave no side-effects.
2. When the parser succeeds, it should only consume the input values required to match the rule. Any extra values read (buffers, lookaheads, etc) should be returned to the input sequence. 
3. Parsers should all expect to be composed. A parser should read input from the current position and not backtrack, and the parser should not necessarily assume that it can read until the end of input. The next parser may continue where the current parser leaves off.
4. The core parsers should, as much as possible, be completely agnostic in terms of input type and output type. Core algorithms should not assume `char` or `string` input.

One big problem with the implementation described in the second blog post is here in this code snippet:

```python
def expression(rbp=0):
    global token
    t = token
    token = next()
    left = t.nud()
    while rbp < token.lbp:
        t = token
        token = next()
        left = t.led(left)

    return left
```

It took me a little while to understand it, in part because I'm not as fluent in Python as I would like to be, and in part because the variables aren't named in a way that describes what they are. The variables `t` and `token` are both token values, for example, but `t` is the current token and `token` is a lookahead value. This parser reads ahead one token, so it can neatly construct the `while` loop condition. Stylistically, I don't like the use of a global variable to hold mutable state. Mechanically, it's harder to make sure we return all the input values that went to create that lookahead `token` back to the input sequence when the parse completes. Dealing with that mechanical problem would require more ugly mutable state.

With a little fiddling, I rearranged the algorithm to not use global state or a lookahead. Instead, I can lean on the `.Rewind()` mechanism, so if I can push ahead with the parse, and then rewind the input when the attempt fails. Here's an approximation of the code I ended up with:

```csharp
public TOutput Expression(ParseState<TInput> state, int rbp)
{
    var leftToken = GetNextToken(state);
    var left = leftToken.NullDenominator();

    while (true)
    {
        var cp = state.Input.Checkpoint();

        var rightToken = GetNextToken(state);
        if (rightToken == null)
            break;

        if (rbp >= rightToken.LeftBindingPower)
        {
            cp.Rewind();
            break;
        }
        var right = rightToken.LeftDenominator(rightContext, left);
        left = right;
    }

    return (true, left.Value);
}
```

It's a little bit less elegant because of the rewinds and the `break` bail-outs from the loop, and I had to trade a little bit of cyclomatic complexity to make it work, but it doesn't require keeping a lookahead value in memory or using global state for anything. Fair trade.

## The Rewrite

A big part of the original motivation for ParserObjects is to make parsers *easy*. I don't just want to implement some parser algorithms for the chosen parsing elites to play with. I want something that has lots of easy on-ramps for developers who might not be familiar with all the theory and jargon. I didn't want to have a Pratt parser implementation, or any parser implementation, if I couldn't make it clean and simple to use. Callback delegates should be typesafe in input and output (and, of course, not assume `char`). In pursuit of this goal, the implementation of the Pratt parser has become a little more complicated than I would like with nested child classes and all sorts of hidden generic type parameters. This extra complexity has paid dividends in terms of usability improvements for downstream users. Here's the updated version of the test from above:

```csharp
var target = Pratt<string>(c => c
    .Add(
        DigitString(), 
        p => p.ProduceRight((ctx, v) => v.Value)
    )
    .Add(
        (Match('+'), Match('-')).First(), 
        p => p
            .LeftBindingPower(1)
            .ProduceLeft((ctx, l, op) =>
            {
                var r = ctx.Parse();
                return $"({l}{op}{r})";
            })
    )
    .Add(
        (Match('*'), Match('/')).First(), 
        p => p
            .LeftBindingPower(3)
            .ProduceLeft((ctx, l, op) =>
            {
                var r = ctx.Parse();
                return $"({l}{op}{r})";
            })
    )
);
var result = target.Parse("1+2*3+4/5");
result.Value.Should().Be("((1+(2*3))+(4/5))");
```

It's more lines of code than the earlier test, to be sure, but there's a clear improvement in expressive power. I don't need a bunch of methods for `.AddInfixOperator` or `.AddPrefixOperator`, both of which are replaced with a single `.Add` method. User callbacks can recurse into the parser easily to get right-hand values to handle all sorts of arbitrary constructs. The new engine also integrates much more cleanly with other `IParser` instances from the library. Here's an example where we parse matching parenthesis, which use the `Match` parser:

```csharp
var target = Pratt<string>(c => c
    .Add(DigitString())
    .Add(
        Match('('), 
        p => p
            .ProduceRight((ctx, op) =>
            {
                var contents = ctx.Parse();
                ctx.Expect(Match(')'));
                return contents;
            })
    )
);
```

You can use existing `IParser`s to call into a Pratt for simple mathematical expressions, you can call other parsers from the Pratt parser. You can mix and match, nest and compose, and build whatever parser graph best suits the needs of your application. It's a much more flexible solution and I'm pretty thrilled with it so far.

## Road Ahead

I'm pretty quickly coming to a "code complete" milestone for the 3.0.0 release of ParserObjects. Most of the remaining TODO issues are about little edge-cases and a few style nitpicks, but no major new features are planned. Right now I'm targetting a release date around the end of December or sometime in early January, depending on how much time I have to work on it.

I had briefly considered adding another parser implementation, for use-cases where the existing combinators and the new Pratt implementation don't quite meet user needs. I was thinking about an [Earley parser](https://en.wikipedia.org/wiki/Earley_parser) or even a [CYK parser](https://en.wikipedia.org/wiki/CYK_algorithm), and I had even briefly considered dabbling in the waters of [PEG parsing](https://en.wikipedia.org/wiki/Parsing_expression_grammar). These are all complicated topics, and I don't have any uses for them myself. Without being a user of these features, there's no way I can properly design them for maximum usability, or give them adequate testing. Plus, I don't have any down-stream uses for these things, yet.

(I actually do have an implementation of an Earley parser in a branch that I have been playing with, but it's ugly right now and needs a lot of tuning and testing. That feature, if I am ever confident enough to merge it, might be ready to go within the next few months, but like I said I don't have any downstream projects driving it so there's very little urgency)

I'm going to spend the next couple days and weeks putting some spit and polish on ParserObjects, especially the new Pratt parser. I'm going to be adding more unit tests and more end-to-end parsing examples to the test suite to show off what's possible in the new system. I'm hoping to have 3.0.0 out the door by the end of December or early January if my current pace of development holds up. If there's one thing COVID quarantine has done well, it's made sure that I have lots of un-interrupted time in front of the keyboard.