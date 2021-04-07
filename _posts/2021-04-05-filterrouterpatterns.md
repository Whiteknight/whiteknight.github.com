---
layout: post
categories: [Design]
title: Filter and Router Patterns
---

As far as I'm concerned, a perfect method has a cyclomatic complexity of 1. It should contain a sequence of statements, read from top-to-bottom, left-to-right, which is as plain to read as it is simple to understand. This doesn't happen much. It's hard to reach perfection. But, it's still a good goal to strive for. The question then becomes, "how do I structure methods which cannot be perfect?" How do we manage and minimize imperfection? Refactoring and iterating are steps along a path, but we need to know what our destination looks like.

**Where control flow primitives are unavoidable, they should at least be made consistent**. This is the thesis of this post. There are two general types or patterns of blocks which I personally try to pursue, and I suggest you do also: The **Filter** and the **Router**.

## The Filter Pattern

The **filter** is a block where all the failure conditions are checked and exhausted before ending up in the one "happy path" of success. Here's an example of a filter loop, from the ParserObjects source code (reduced down, for readability):

```csharp
while (true)
{
    var rightResult = GetRight(state, rbp, leftToken);
    if (!rightResult.Success)
        break;
    if (rightResult.Value == null)
        break;
    if (rightResult.Consumed == 0)
        break;

    consumed += rightResult.Consumed;
    leftToken = rightResult.Value;
}
```

This code is part of the *Pratt parser engine*, though I won't go into detail here about what it does and why. What's important is the shape of the code: it is a filter. On each iteration of the loop it tests `rightResult` for a bunch of failure conditions and breaks from the loop if any check fails. It's only when we exclude all failure possibilities that we continue along the happy path of success and allow the loop to continue. Here's another example from the same source:

```csharp
foreach (var parselet in _ledableParselets.Where(p => rbp < p.Lbp))
{
    var (success, token, consumed) = parselet.TryGetNext(state);
    if (!success)
        continue;

    var rightContext = new ParseContext<TInput, TOutput>(state, this, parselet.Rbp, true);
    var rightResult = token.LeftDenominator(rightContext, leftToken);
    if (!rightResult.Success)
        continue;

    return new SuccessPartialResult<IToken<TOutput>>(rightResult.Value);
}
```

Again, the exact purpose of this code isn't nearly as important as it's being a demonstration of a filter. In this case the `return` statement at the bottom of the loop is the happy path which breaks out of the loop. Every other non-success condition leads to a `continue` which repeats the loop until a success is found.

"Success" means something different in each context. Sometimes success is continuing the loop. Sometimes success is finally breaking out of a loop. In many cases there are no loops at all, just a bunch of `if/return` or `if/throw` checks before getting on with the business at hand. In any case, a filter should test for and rule out all the causes for failure first, only to finally arrive at the single successful result at the end. At that point, once we know that the preconditions are all satisfied, we can operate on the data with high confidence.

Filters are not limited to just loop control. Methods can be filters as well. In fact, here's a particularly short method example:

```csharp
public void TryAdd(object value)
{
    if (_collection.Contains(value))
        return;
    _collection.Add(value);
}
```

The lone failure condition here is that the collection already contains the given value so it cannot be added. The success happy path is that the value is added to the collection, after we've checked first that it's possible to do. Do we need to add more precondition checks? Do it at the top. Do we need to add more processing steps to the happy path? Do it at the bottom. Further, we know that we aren't leaving work partially done. We don't start doing our processing until we've covered all the error cases, which means we don't need to rollback partial work.

Another way to structure this is using LINQ methods to filter out unusable data:

```csharp
return inputData

    // The filter
    .Where(ExcludeFailureConditionA)
    .Where(ExcludeFailureConditionB)
    .Where(ExcludeFailureConditionC)

    // The happy path of success
    .FirstOrDefault();
```

Whatever form you choose to write it in, the salient part of the filter is that all error conditions are tested and excluded first, before successful outcomes are evaluated. Once all the possible error conditions are excluded, the remainder of the code can operate with confidence that data is in correct formats and preconditions are met. 

## The Router Pattern

The **router** is a block where many possibilities for success are attempted before ending up in a final error condition. It is the opposite of the filter. We exhaust all of our possibilities for success before finally giving up. Many `switch` and pattern-matching statements tend to work this way:

```csharp
 public IResult<TOutput> Parse(IParseState<TInput> state)
{
    return _quantifier switch
    {
        Quantifier.ExactlyOne => ParseExactlyOne(state),
        Quantifier.ZeroOrOne => ParseZeroOrOne(state),
        Quantifier.ZeroOrMore => ParseZeroOrMore(state),
        _ => state.Fail(this, $"Quantifier value {_quantifier} not supported"),
    };
}
```

In this block we test all the different valid values for `_quantifier` before finally falling back to the default, unmatched option which leads to a failure. As with filter above, router isn't limited to just one implementation. You could have a router method, or a loop which tries to route successes before moving on to some kind of failure.

## What's The Alternative?

Many programmers of a certain generation subscribe to the idea that a method should have one entry point and one exit point. Methods constructed under this kind of philosophy tend to be deeply nested with lots of `if`/`else` blocks, all setting some kind of `result` value which is eventually returned. The problem with methods constructed this way is that it's never clear where control flow should be going. What does failure look like? What does failure recovery look like? When we're debugging, how do we know which path we're supposed to be taking to get to the end? In a filter or router, the answers to these questions are obvious.

Other methods will not have the deeply-nested structure of single-return methods, but instead will use early bail-outs where appropriate to reduce nesting and reduce the need for mutable `result` variables with large scope. While better, these methods are not necessarily great. Do a check and return a success, then do another check and return failure. Another check and another check, sometimes returning one and sometimes returning the other. The problem with methods like this is that it's still impossible to know what the intended path is through the method, and it's impossible to reason about anything without having everything crammed into your head-space.

Again, I'm talking about these as goals to strive for, and it's not always possible to get there for every method you write. I have lots of methods in my own project which I couldn't quite structure as filters or routers, and readability definitely suffers in these because of it. Some complexity is necessary, just make sure you justify it.

## Why Filter and Router?

The thing with Filter and Router patterns is that they tend to be quite easy to read and reason about. In the case of filter, it's easier to put all your validation logic up front, and even pull validation out into a separate method or object if necessary, and know that when you are working on the happy-path code that you have assurance against common errors. In the case of the router, it's nice to be able to clearly list all the possibilities for success, and see clear logic which leads to each successful continuation. You only resort to failure when all possibilities for success have been exhausted.

Exactly what "failure" represents in each case is contextual. Sometimes you'll return a default or `null` value, or use the *Null Object Pattern*. Sometimes you'll throw an exception. In many cases, you may want to accompany these failure conditions with logging for debugging purposes.

If you know that there is a clear delineation between the parts of the method which lead to success from those parts which lead to failure, it is easier to focus on the part which needs your attention. In the case of the filter, it's about explicitly listing all the things you don't want, then working with whatever is left over. In the case of the router, it's about explicitly listing all the things you do want, before communicating an error. 

You can even combine the two. First you use a filter to remove all sorts of spurious and malformed inputs, then you call into a router to specifically handle whatever is left over. Or do it the other way around: A router to handle all the easy cases, then fall back to a filter where you deal with obvious errors and try to salvage something usable in the end. These things are flexible, and they're great for composition.

This is just one way I try to separate my code. I also try to separate pure and impure code. And also [orchestrators from non-orchestrators](http://whiteknight.github.io/2015/06/19/whennottounittest.html). 
These things all overlap, and sometimes framing a problem in terms of different sets of alternates helps to understand it better.