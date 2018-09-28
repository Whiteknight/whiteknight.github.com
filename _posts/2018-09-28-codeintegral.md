---
layout: post
categories: [Design, Theory]
title: Code Integrals
---

Trying to figure out what is good code and what is bad code is quite a difficult yet important task. We'd like to know that when we add a new feature that we haven't significantly decreased the quality of the code we're working on. We'd also like to be able to show that when we refactor and attempt to improve code, that we've actually produced a benefit. There are a couple rules of thumb and concepts that I generally try to follow, and that I've heard at least some other programmers following as well:

1. A method should fit entirely in a single screen of your editor
1. A line of code should be no longer than 80-100 characters long
1. The Single Responsibility Principle
1. Cyclomatic Complexity should be as low as possible and absolutely never above some threshold (say, 10-20)

I've been kicking around an idea to augment these other principles that I'm calling, for lack of a better name, the **Code Integral** or **Integral Complexity**. It's named after the humble mathematical integral, which is used to calculate the area under a curve.

## In a Nutshell

The simple version of it is this: We want to measure the "area under the curve" where the curve is the indenting of your code and the area is the whitespace to the left of it. We start by figuring out where the base indentation for the method *should be*. This is position 0 for our calculation and all lines of code at that base indentation add 0 to our result. For every line of code in the method, we count the number of standard indentations from the base point, and add that to our running total. Let me show a few examples.

First, a quick aside about my personal style: My "standard indentation" is 4 spaces, and I'm using Allman-style braces. A line with just a brace doesn't count as a "line of code" so it doesn't get added into the result. So, in this scheme, the "base indentation" is 4 spaces in from the column where the method is defined (which, in standard C# is frequently indented 4 spaces from the class definition which is intended another 4 from the namespace definition, for a base position of column 12.) In the examples below, I'll show the area under a line in a comment to the right of it.

```csharp
public void FirstExampleMethod()
{
    DoOneThing();       // 0
    DoAnotherThing();   // 0
}
```

Here, `FirstExampleMethod` has integral complexity of 0. All the lines of code are at the base indentation. You'll notice that this also has a cyclomatic complexity of 1, which is the lowest in that particular measurement.

```csharp
public void SecondExampleMethod()
{
    if (CheckPrecondition())    // 0
    {
        DoOneThing();           // 1
        DoAnotherThing();       // 1
    }
}
```

`SecondExampleMethod` has an integral complexity of 2. It has 2 lines which are intended one position each.


```csharp
public void ThirdExampleMethod()
{
    if (!CheckPrecondition())   // 0
        return;                 // 1

    DoOneThing();               // 0
    DoAnotherThing();           // 0
}
```

`ThirdExampleMethod` performs the exact same work as `SecondExampleMethod`, but by inverting the condition we reduce the overall level of indentation and lower the integral complexity to 1. In other words, this simple isomorphic transformation of the code has decreased the integral complexity and, according to me, improved it. Now let's look at an example a bit more complicated:


```csharp
public void FourthExampleMethod()
{
    var batches = Items.GroupIntoBatches(); // 0
    foreach (var batch in batches)          // 0
    {
        BeginBatch(batch);              // 1
        foreach (var item in batch)     // 1
        {
            DoAThing(item);             // 2
            DoAnotherThing(item);       // 2
            DoAThirdThing(item);        // 2
        }
        CompleteBatch(batch);           // 1
    }
}
```

In `FourthExampleMethod` we have integral complexity of 9, and cyclomatic complexity of (if I am doing my mental math right) about 5 or 6, and are starting to create an ugly [Arrow Antipattern](https://blog.codinghorror.com/flattening-arrow-code/). We can refactor this down to minimize the integral complexity, by pulling some of the code out into dedicated methods:


```csharp
public void FifthExampleMethod()
{
    var batches = Items.GroupIntoBatches();     // 0
    foreach (var batch in batches)              // 0
        ProcessBatch(batch);                    // 1
}

private void ProcessBatch(Item[] batch)
{
    BeginBatch(batch);          // 0
    foreach (var item in batch) // 0
        ProcessItem(item);      // 1
    CompleteBatch(batch);       // 0
}

private void ProcessItem(Item item)
{
    DoAThing(item);         // 0
    DoAnotherThing(item);   // 0
    DoAThirdThing(item);    // 0
}
```

Now in `FifthExampleMethod` we can see we have a total integral complexity of 2, which is a significant improvement over the value of 9 from the `FourthExampleMethod`. Again, I generally posit that a lower integral means better code.

### Better?

It's worth asking, at this point, what "better" code is. Code is the exercise of trying to give instructions to a computer in a way that other programmers can easily read and understand. It's often easy to favor one over the other: Some code is great for the computer but hard for the human, and other code is great for the human and very difficult for the computer. Great code tries to maximize ease for both.

Decreasing the integral of the code means fewer control structures and fewer lines of code nested into those control structures. This, in turn, decreases the mental effort of the programmer to understand. The best code, with an integral of 0, can simply be read from top to bottom as a list of steps to execute, requiring almost zero effort on the part of the programmer to follow. Likewise, things might be better for the computer as well, with the happy path being easy to follow and the code playing nice with the branch predictor and cache.

The worst code, with deep nesting of control structures, can be extremely difficult for humans to follow, which in turn slows down development and raises the risk of bugs and other problems.

I've said before and I'll say many more times over the course of my career: The most important use of a method is to give a block of code a descriptive name. In the `FifthExampleMethod` above, we break one method down into three, with the two new methods having names that, while short, tell exactly what they do: The first processes a single batch, the second processes a single item from that batch. In addition to decreasing the integral of the code, we've also clearly delineated what kinds of operations occur in which places. Operations on the batch happen in the `ProcessBatch` method and operations on the individual item happen in the `ProcessItem` method. The job of the next programmer to work on this code is thus simplified significantly.

Decreased integral does not necessarily correlate to better code, but it's been a very powerful heuristic for me to follow in developing my own code and improving code that I am maintaining.

## Comparison with Cyclomatic Complexity

[Cyclomatic Complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity) counts the number of distinct code paths, which typically correlates closely with the number of unit tests required to properly cover the code. Code integral, on the other hand, has no obvious relationship to unit testing.

Code integrals can easily be summed up over a class, a module, or even an entire program, because the base number of each method is 0. Adding many zeros together still gives 0. Cyclomatic Complexity, on the other hand, has a minimal value of 1, so adding it up over a class, module or program can give skewed and nonsensical results. A simple method with no control structures has 0 integral complexity but 1 cyclomatic complexity, which seems to deter creating such methods even if the grouping of statements provides other benefits. Consider these two examples:

```csharp
public void PerformTask()
{
    var t = GetWorkTask();
    t.Validate();
    t.Prepare();
    t.Execute();
    t.Cleanup();
}
```

Compared to this where we decompose into two methods:

```csharp
public void PerformTask()
{
    var t = GetWorkTask();
    ExecuteTask();
}

private void ExecuteTask(WorkTask t)
{
    t.Validate();
    t.Prepare();
    t.Execute();
    t.Cleanup();
}
```

Most automated tools I've seen would give the first example a cyclomatic complexity of 1, but give the second example a 2 (1 for each method) despite the fact that there is exactly one path that the instruction pointer can take through this code. Cyclomatic complexity either cannot be summed over an entire class or module, or else we find ourselves wanting to have fewer descriptively-named methods. VisualStudio is an example of a tool like this and it annoys me every time I use it to calculate code complexity.

Code integral promotes or supports several common programming idioms, such as bailing out early with precondition checks and guard clauses (Defensive Programming), decomposing larger methods into smaller methods, etc. In fact, most of the common named refactorings either do not affect the code integral or serve to improve it. It avoids some common problems like the Arrow Anti-Pattern. Cyclomatic complexity, on the other hand, doesn't directly promote any good idioms or avoid any bad ones.

In general the code integral will go up as the cyclomatic complexity increases, though I wouldn't call them proportional per se. The rates of growth of the two measures will increase at different rates as lines of code and control structures are added. Adding more lines of code to existing control structures may increase the integral but will not affect the cyclomatic complexity, and a method with a very low cyclomatic complexity can still be flagged by an integral checker as needing refactors.

## Difficulties and Heuristics

An obvious concern is that code integral becomes highly dependent on code formatting. Putting multiple constructs on a single line has the effect of making code quality worse while also decreasing the apparent integral. To avoid that, we need standard formatting rules and, likely, automated linting. Consider a method chain:

```csharp
public void SixthExampleMethod()
{
    obj.DoThingA().DoThingB().DoThingC().DoThingD();
}
```

A heuristic that I've been having trouble with is trying to have a single method on a single line, because the line above in `SixthExampleMethod` can easily become quite hard to read. Moving methods onto separate lines, especially where we've got a long method chain, can improve readability by a significant amount:

```csharp
public void SeventhExampleMethod()
{
    obj.DoThingA()
        .DoThingB()
        .DoThingC()
        .DoThingD();
}
```

But then again, this code in `SeventhExampleMethod` seems like it has an integral of 3 instead of the 0 which the original version has. Consider this next version, which is equivalent, but would arguably be considered much more complex because of the addition of so many more explicit state variables (especially if, in the below example, those temp variables don't get descriptive names):

```csharp
public void EighthExampleMethod()
{
    var a = obj.DoThingA();
    var b = a.DoThingB();
    var c = b.DoThingC();
    c.DoThingD();
}
```

We don't want people to be able to game the system by making things *less readable*, so I haven't quite figured out how to handle this case of method chaining.

Likewise, large boolean expressions with lots of `||` and `&&` clauses can quickly become unreadable, and so could benefit from being broken up onto multiple lines. The same can be said about large arithmetic expressions. The common refactoring [Decompose Conditional](https://refactoring.com/catalog/decomposeConditional.html) can be used to pull complex conditionals into separate, descriptively-named methods, and similar refactoring can give a nice name to complicated arithmetic. The problem I run into is that a long expression like this *should be considered more complicated* than a single line of normal code, and the integral should account for that. Imagine, if you will, that `SeventhExampleMethod` above were a LINQ method chain with parameters and `Func` delegates all over the place. A complicated method chain like that should increase a complexity score. Cyclomatic Complexity doesn't generally deal with this (though some implementations do account for lambdas).

Maybe we can ignore long individual lines of method chains and complicated arithmetic for the integral metric, but then we would still want some kind of way to flag these kinds of lines for review. I have in mind automated tools that would throw up a warning at build time (if not earlier) to say "This method has an integral of 50, consider refactoring it" or "This method has a very long and complicated line of code which should be decomposed". I'd love it if both these things could be covered by the same analysis tool, but that might be a bit too ambitious.

In either case, using a good and standard linting tool across the entire team will help with calculating the integral in a consistent way.

Some other areas to consider are:

1. Breaking a long list of method parameters or method arguments onto multiple lines probably won't add to the integral, though large parameter lists are generally problematic.
1. Declaring a variable without initializing it to a value likely shouldn't be counted towards the integral, or not at full price.
1. Comments and empty lines don't count

## General Use

When I'm coding I rarely calculate integral complexity explicitly, but I always keep an eye on the whitespace on the left side and work to reduce it when I am refactoring. I find that the resulting code is frequently better as the integral decreases: more readable, more maintainable, less complicated.

Considering my earlier posts about [unit testing](2015-6-19-whennottounittest.md) and [thin controllers](2015-6-23-thingcontrollers.md), I can make a few statements with a bit more rigor:

1. An ideal MVC or WebApi **Controller** class should have a total integral of 0.
1. An ideal **Service Layer** class should have a total integral of 0.
1. The number of dependencies a class has should be inversely proportional to the integral of the class. That is, if the class has many dependencies, it should have a very low integral complexity so we don't need to unit-test it as vigorously. A single class should either do a lot of work or coordinate/delegate a lot of work, but not both.
1. Any method with an integral complexity of 0 and where most of the work is delegated to other methods or classes almost never needs to be explicitly unit tested.

## Combined Measure

The code integral isn't a final determination about code quality, and it often feels like just the first input to a larger metric. For example, combining the code integral with a count of method parameters and a count of local variables would likely be a good idea. It might even be worth counting the total number of operators (arithmetic, logical and even the method-invoking `.`). Decomposing a long method chain by introducing a lot of temporary variables, the example I showed above, is clearly not a huge improvement and a combined measure where we counted the number of such variables would show that.

What we're really trying to get to is a simple flag: "Yes this method needs some refactoring attention" or "No this method is not an immediate source of concern". So the question isn't "what is the integral?" but instead "is the integral above a certain threshold?". A warning coming out of a compiler or a big red exclamation point in a code review tool is what we want to see.

## Conclusion

So this is my idea for "Integral Complexity" or "code integral" or whatever I call it. I've been using it in an informal way in my own programming for a while now and have been quite happy with the results. I would eventually like to put together an automated tool to calculate it (and related measures), but I haven't gotten that far yet. If anybody else has any feedback about this idea, I'd love to hear about it.