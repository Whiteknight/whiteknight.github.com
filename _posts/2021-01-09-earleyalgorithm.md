---
layout: post
categories: [projects, algorithms]
title: The Earley Algorithm
---

The [Earley algorithm](https://en.wikipedia.org/wiki/Earley_parser) is a powerful and efficient matching technique, which I understand is pretty well-respected in academia. 

The recursive descent algorithm, on which the [ParserObjects]() parser-combinators approach is built, is greedy and depth-first; handling ambiguities in the grammar by usually ignoring them. It also has some significant challenges with left-recursion which some extensions help with but do not solve completely. The Pratt algorithm is likewise based on recursive descent and has many of the same drawbacks, though is better at handling different types of associativity and precedence. On the other side of the scale, recursive descent (especially combinators) are quite easy to use. Almost any Joe Schmoe programmer with a tutorial in one hand and a keyboard under the other can put together a reasonable recursive descent parser to parse a lot of real-world languages and grammars. These parsers can also be pretty easy for a person to read and understand without a huge amount of prior training in parsing theory.

The Earley algorithm, on the other hand, is a parallel breadth-first algorithm which handles ambiguity by keeping track of all possibilities in the parse simultaneously. Instead of returning a single parse tree, the Earley parser returns an entire **parse-forest** with all possible matching results. The Earley algorithm is also a lot more powerful, able to handle left- and right-recursion with relative ease compared to the other approaches I mentioned. In exchange for all this power and theoretical awesomeness, there are a few problems with the Earley algorithm: First, it is a significantly more complicated algorithm to understand and implement than combinators or Pratt, and it is relatively slow and difficult to derive results from a successful match. 

There also hasn't been, as far as I can tell, a lot of research or prior art involved in integrating the Earley algorithm with combinators. Assuming that the ultimate goal of combinators is to make parsers composable and reusable, and the fact that sub-grammars very well may be left-recursive or ambiguous, it seems like this is something worth exploring, at least.

I decided that I wanted to start working on just such a project, to bring the power of Earley into the ParserObjects library. Like my Pratt implementation, I wanted it to be fully integrated: Combinators should be able to call Earley to handle sub-grammars, and the Earley engine should be able to delegate matching out to combinators. I also want it to have decent performance and to be very easy to read and maintain. Also I would like a pony, a million dollars, and world peace.

## The Algorithm, Quick and Stupid Overview

There are resources online to understand how the Earley algorithm works, so I won't go into a heck of a lot of detail here. I'll try to briefly touch on the parts which are salient for my immediate purposes.

The algorithm centers on the use of the **Earley Item**, which is a combination of a production rule, a reference to the position where the rule started, and the current state of matching the rule. So if we have the following production rule:

```
publicModifier := 'p' 'u' 'b' 'l' 'i' 'c'
```

An Earley Item for this production might look like this:

```
(publicModifier := 'p' ● 'u' 'b' 'l' 'i' 'c', 4)
```

In this Item the ● "fat dot" shows the current state of the partial match. Everything to the left of the dot has been seen already, everything to the right is yet to be seen. The `4` there shows that the item started at input position 4.

Items are organized into State Sets. A State Set represents a single position in the input string, and contains all items which are possible or in-progress at that position. Using the example from above, Set `S(5)` has just seen a `'p'` character, so `S(5)` may contain items like the following (and several others, not shown):

```
(publicModifier   := 'p' ● 'u' 'b' 'l' 'i' 'c', 4)
(privateModifier  := 'p' ● 'r' 'i' 'v' 'a' 't' 'e', 4)
(identifier       := 'e' 'x' 'p' ● <letters>, 2)
```

The basics of the algorithm work as follows: We start at the initial State `S(0)` which has consumed no input (the Fat Dot is at the far left for all rules) and has a single top-level rule to try and match, we iterate over states as follows (again, I'm glossing over a lot of detail):

1. Loop over all Items in the current State `S(n)`
   1. **Scan**: If the next symbol in the Item is a Terminal (represents a single input value to be matched), try and match the Terminal to the current input value. If it matches, add a new Item to the next State `S(n+1)` which is the same as the current item, but with the Fat Dot moved one position to the right. This new Item maintains a reference to the "parent Item" from which it came. 
   2. **Predict**: If the next symbol in the Item is a Non-Terminal (represents several things which need to match in order), add Items to the current State `S(n)` for each sub-rule which needs to be matched at the current position.
   3. **Complete**: If the Item is completed (the Fat Dot is all the way to the right), go through Items in the Parent State `S(parent)`, adding any Items items which were waiting for the current Item (the Fat Dot was immediately to the left of that part of the rule) to the current State `S(n)`, with the fat dot moved forward one position. 
2. When we have reached the end of input, check to see if we have any Items in the final State which have `S(0)` as the Parent State, are for the Top-Level production rule, and are Complete. If so, break and return these as the final results.
3. Otherwise move to the next State `S(n+1)` and repeat.

Basically, every Item which consumes actual input moves to the next State, otherwise we keep looping over the current state trying to simplify rules until we either find an item which consumes input, find a final result, or run out of things to simplify. There's a lot more to it which, again, I'm glossing over intentionally. If you want to learn more, a great tutorial on the process is [in this blog post here](https://loup-vaillant.fr/tutorials/earley-parsing/) and there's some (very outdated) example code [in this github repo](https://github.com/dominiccooney/earley), which I referenced a lot to get started on my own implementation.

At the end of the algorithm, when we come to the end-of-input and have our list of result Items, we need to derive results. This is where things get a little messy. We have to visit the tree of result Items, building up derivation values from each Item in turn for all the possible parse values which that Item may contain. This process is a little ugly and a little messy, and I won't talk about it any more here.

There are also a few other important details to talk about, though maybe not in this post: The [Aycock/Horspool fix](https://pdfs.semanticscholar.org/5b27/7a3f9a9f5250939f334e282d1393971722a9.pdf), the [Leo optimization](https://www.sciencedirect.com/science/article/pii/030439759190180A), and the [Scott SPPF Algorithm](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.1049.562&rep=rep1&type=pdf), for example. I won't discuss those here either, but suffice to say it's hard to talk about an Earley implementation being "complete" and "correct" without these things.

## Quick Example

Let's say we have a basic expression grammar without operator precedence:

```
E := number
E := E '+' E
E := E '*' E
S := E end
```

Here `S` is our top-level start rule. Notice that `E` is both left- and right-recursive. Running the algorithm on the input string `"4*5+6"` will produce a series of Items like this:

```
==== S(0) ====
S    := * <Expr> <End>  (0,0)
Expr := * <literal>  (0,0)
Expr := * <Expr> <'+'> <Expr>  (0,0)
Expr := * <Expr> <'*'> <Expr>  (0,0)

==== S(1) ====
Expr := <literal> *  (0,1)
S    := <Expr> * <End>  (0,1)
Expr := <Expr> * <'+'> <Expr>  (0,1)
Expr := <Expr> * <'*'> <Expr>  (0,1)

==== S(2) ====
Expr := <Expr> <'*'> * <Expr>  (0,2)
Expr := * <literal>  (2,2)
Expr := * <Expr> <'+'> <Expr>  (2,2)
Expr := * <Expr> <'*'> <Expr>  (2,2)

==== S(3) ====
Expr := <literal> *  (2,3)
Expr := <Expr> <'*'> <Expr> *  (0,3)
Expr := <Expr> * <'+'> <Expr>  (2,3)
Expr := <Expr> * <'*'> <Expr>  (2,3)
S    := <Expr> * <End>  (0,3)
Expr := <Expr> * <'+'> <Expr>  (0,3)
Expr := <Expr> * <'*'> <Expr>  (0,3)

==== S(4) ====
Expr := <Expr> <'+'> * <Expr>  (2,4)
Expr := <Expr> <'+'> * <Expr>  (0,4)
Expr := * <literal>  (4,4)
Expr := * <Expr> <'+'> <Expr>  (4,4)
Expr := * <Expr> <'*'> <Expr>  (4,4)

==== S(5) ====
Expr := <literal> *  (4,5)
Expr := <Expr> <'+'> <Expr> *  (2,5)
Expr := <Expr> <'+'> <Expr> *  (0,5)
Expr := <Expr> * <'+'> <Expr>  (4,5)
Expr := <Expr> * <'*'> <Expr>  (4,5)
Expr := <Expr> <'*'> <Expr> *  (0,5)
Expr := <Expr> * <'+'> <Expr>  (2,5)
Expr := <Expr> * <'*'> <Expr>  (2,5)
S    := <Expr> * <End>  (0,5)
Expr := <Expr> * <'+'> <Expr>  (0,5)
Expr := <Expr> * <'*'> <Expr>  (0,5)
S    := <Expr> <End> *  (0,5)
```

(This output comes almost verbatim from my current implementation, I didn't write this out by hand. This is why it uses the asterisk instead of the "●"). I won't go over this whole thing line-by-line, but notice that in the final State `S(5)` there are these three important Items:

```
Expr := <Expr> <'+'> <Expr> *  (0,5)
Expr := <Expr> <'*'> <Expr> *  (0,5)
S    := <Expr> <End> *  (0,5)
```

The last Item here, which is the end of the start rule, represents the completed result. The first two Items are the two possible results for `Expr`. Without specifying precedence levels, this parser found two variants: `(4 * 5) + 6 = 26` and `4 * (5 + 6) = 44`. Here's the test from the ParserObjects test suite which does this:

## Integration Into ParserObjects

It's worth reiterating some of the invariants of the parser objects library:

1. On failure the parser should return the input sequence to the state it was in before the parse was attempted.
2. On success the parser should only consume the input values that were used for the match, and should return any other lookaheads or buffered values back to the input sequence
3. Parsers should read input starting from the current location only, and should not try to explore what input values had been consumed previously.

Like with Pratt, there are some changes to be made to the Earley algorithm so it can play nicely with the the ParserObjects approach and the invariants listed above. First, a Terminal in the Earley parser wouldn't just be matching a single input character at a time, it would be delegating to an arbitrary `IParser` implementation which may consume many input items, or may even consume nothing but still report success (an empty list, the Empty rule, an Optional rule which didn't work out but still reports success, the End rule, etc). The Earley engine also needs to play nicely with the `ISequence` input type, which streams input from a source that is not necessarily a fixed buffer. If Earley calls out to something simple like `Match('+')` that's pretty simple, and the algorithm can remain unchanged. But if it calls out to something slightly more complicated like `Match("!==")` we need to put the successful result into State `S(n+3)` instead of `S(n+1)` because we have consumed more than 1 unit of input. If we have finished processing `S(n)` and `S(n+1)` is empty but `S(n+m)` has items, we have to skip to `S(n+m)` (and hopefully do it in a way that isn't `O(m)` in time complexity). We also have to be aware that the Earley parser might not consume all remaining input, but instead may terminate early. This means we need some way to signal completion besides waiting for the end of input.

And when we complete, we need to be able to return all possibilities, including those which were shorter and those which are longer. Consider a grammar represented by this regex:

```
Foo(Bar)?
```

In this case a successful result might be `"Foo"` with a length 3, and also `"FooBar"` with length of 6. Ideally we would like to return both possibilities, which means we need to find result Items in all States, not just the final State. Also, we need a way to select which result to use to continue parsing, and we also need a way to rewind the input to the location of the end of the result, so that the next parser can continue from the correct position. Just getting results out involves a lot of modifications and extensions to the original algorithm.

There are also a few other details: Each loop iteration cannot advance the input manually, but must instead rely on delegated `IParser`s to advance it. After each match attempt, the input sequence must rewind to the position specified by the current state. Each Item must include a count of how many input items it consumed and also be able to return input to the position from the end of the match, so the next symbol can continue from the correct position.

To reiterate a few points, Earley Parser should:

1. Delegate to other existing `IParser` objects to match and consume input, which may match input sequences of varying length.
2. Not necessarily consume all remaining input, but instead will try to consume as much as possible.
3. Be able to return all possible matches, including those of different lengths
4. Be able to select the "best" result among all possibilities, so that the rest of the parser graph can continue the parse from that location.

## Derivation

I glossed over Derivation earlier. The way it works, turning a match into a parse, is that we traverse the tree list of Items from the final result Items and their children rules, invoking callback delegates at each to produce a value for that Item. We start at the bottom of the tree and work our way up, maintaining a list of all possibilities at each step. The number of possible results grows as the number of possibilities at each step increases. For example, given this recursive production rule

```
Expr := Expr '-' Expr | number
```

When we parse an input like `4-5-6-7`, the two recursive calls to `Expr` could be (`4`, `5-6-7`), (`4-5`, `6-7`), or (`4-5-6`, `7`) giving three options. And, of course, each recursive call could itself have multiple options. I don't know exactly what the mathematical expression is for how large the number of possible results grows in the parse forest, but I know it can be significant.

Integrating this piece into ParserObjects is a little more tricky, because derivation rules are implemented with user-callbacks, and users will use those callbacks to do things like signal failure or even throw exceptions. This is the only real way to communicate failure distinctly from an empty set of results. All the matches which didn't work out are silently dropped by the algorithm, but errors from user-callbacks will create error results.

## Current Status

My first-draft of an Earley implementation is completed and the branch has been pushed to github. Here are two tests from the test suite showing some of what's possible (the first of these generated the output listed above):


```csharp
 [Test]
public void BasicExpression_EOF()
{
   var target = Earley<int>(symbols =>
   {
         var plus = Match('+').Named("'+'");
         var star = Match('*').Named("'*'");
         var literal = UnsignedInteger().Named("literal");

         var expr = symbols.New("Expr");
         expr.AddProduction(literal, n => n);
         expr.AddProduction(expr, plus, expr, (l, _, r) => l + r);
         expr.AddProduction(expr, star, expr, (l, _, r) => l * r);

         var eof = If(End(), Produce(() => true));

         return symbols.New("S").AddProduction(expr, eof, (e, _) => e);
   });

   var result = target.Parse("4*5+6");
   result.Success.Should().BeTrue();
   result.Results.Count.Should().Be(2);

   var values = result.Results.Where(r => r.Success).Select(r => r.Value).ToList();
   values.Should().Contain(26); // (4*5)+6 = 26
   values.Should().Contain(44); // 4*(5+6) = 44
}
```

This second example shows one way that I am (currently) allowing the Earley parser to chain results to other combinators:

```csharp
[Test]
public void RightRecurse_ContinueWith()
{
   // E ::= a | a E
   var target = Earley<string>(symbols =>
   {
         var a = Match('a').Named("a");

         var e = symbols.New("E");
         e.AddProduction(a, _ => "a");
         e.AddProduction(a, e, (_, rr) => "a" + rr);
         return e;
   });

   var parser = target.ContinueWith(left =>
         Rule(
            left,
            Any().ListCharToString(),
            (l, rem) => $"({l})({rem})"
         )
   );

   var result = parser.Parse("aaaaa");
   result.Success.Should().BeTrue();
   result.Results.Count.Should().Be(5);

   var values = result.Results.Where(r => r.Success).Select(r => r.Value).ToList();
   values.Should().Contain("(a)(aaaa)");
   values.Should().Contain("(aa)(aaa)");
   values.Should().Contain("(aaa)(aa)");
   values.Should().Contain("(aaaa)(a)");
   values.Should().Contain("(aaaaa)()");
}
```

Notice that the Earley parser parses all the possible non-empty prefixes of repeated `'a'` characters, and the `Any().ListCharToString()` parser consumes all possible suffixes. The Earley parser produces multiple results, and each result is piped into another parser to produce another list of results. Later you can apply some kind of selection criteria to return just a single "best" result (whatever "best" means to you) or you can throw an error if there are more than one successful results indicating an ambiguity in the grammar. There's going to be a lot of flexibility here, and I'm excited about it.

## Going Forward

I'm still very much in an exploratory phase with Earley right now, making sure it is both easy to use and sufficiently performant. There's a branch for `v3.1` which is already starting to get some new features like memoization, unique Ids for parsers, and some ease-of-use improvements. When I'm happy and confident enough with the Earley implementation, I'll merge that into the `v3.1` branch and start really cleaning it up and expanding test coverage. 

There's also another algorithm that I have been exploring, [GLL](http://www.cs.rhul.ac.uk/research/languages/csle/GLLparsers.html). I have only started sketching out the basics of a GLL implementation so far, but if that goes well I might like to think about merging it into the `v3.1` branch as well. Only time will tell if I can complete both implementations, and if I am happy enough to merge one or both or neither.