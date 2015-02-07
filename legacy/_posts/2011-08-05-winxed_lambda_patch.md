---
layout: post
categories: [Parrot, Winxed]
title: Winxed Lambda Syntax Patch
---

NotFound is the author of Winxed, the lowish-level Parrot language that I
use the most. I find it's much closer to the underlying machine than NQP-rx
is, and when you're a core developer developing core components, that kind
of closeness and predictability is a big plus. I won't get into a big
discussion here about the relative merits of Winxed and NQP, or any other
languages, I'm only pointing out that when I need to write code for execution
on Parrot, I prefer to use Winxed for most cases.

Because I use it so often, I end up generating lots of feedback for NotFound.
To his credit, he has a pretty strong vision of what the language should be,
and where my feedback conflicts with his goals, he does things his own way.
That's a good thing. Clarity and consistency of vision is an important thing
for any project, programming languages especially.

Two days ago I suggested to him that a more concise syntax for creating
anonymous subroutines and closures would be a big benefit. At least, it would
be a nice benefit for me because I end up writing lots of them as part of
Rosella, and I end up using lots and lots of them in the examples and tests
for Rosella.

He expressed some concerns that the new syntax, in addition to the old syntax,
would be a confusing duplication, and the meaning of a shorter syntax would
not be obvious or particularly readable for new users of the language. And,
considering that Winxed itself is a new language, almost all users of it will
be new users. Here's the current syntax for creating an anonymous subroutine
in winxed:

    var f = function() { say("hello!"); };
    f();

After a surprisingly short and easy hacking session last night, I added a
proof-of-concept new syntax to the parser that looks like this:

    var f = -> { say("hello!"); };
    f();
    var g = -> (x) { say(x); };
    g("hello!");

With the new syntax, if we don't have any parameters we can leave out the
parenthesis entirely. Basically, the new syntax allows us to change the
'function' keyword with the operator '->', and optionally omit the
parenthesis. It's an improvement, but not super-duper. Because Winxed allows
hash-literals to be defined with curly brackets, we can't have a syntax of
just brackets, like what NQP can do. In NQP-rx:

    my &f := { pir::say("Hello!"); };
    &f();

NQP-rx can get away with just the brackets to indicate a code literal, because
it doesn't conflict with a syntax for hash literals. Winxed does have the hash
literals syntax so it cannot have unambiguous simple closures like this. It's
not really much of a loss, the two extra characters to type, "`->`" really
isn't much overhead, and it's still less than writing 'function'.

To see the benefit in readability of the new syntax, consider something like
an example from the Rosella Query library documentation:

    var data = [1, 2, 3, 4, 5, 6, 7, 8, 9];
    using Rosella.Query.as_queryable;
    int sum = as_queryable(data)
        .map(function(i) { return i * i; })
        .filter(function(j) { return j % 2; })
        .fold(function(s, i) { return s + i; })
        .data();
    say(sum);   # 165

We can rewrite that middle part as:

    int sum = as_queryable(data)
        .map(->(i) { return i * i; })
        .filter(->(j) { return j % 2; })
        .fold(->(s, i) { return s + i; })
        .data();

I think this is a lot cleaner and more readable. It isn't as thin as it
could be, we're still typing a few extra characters for situations where we
really just want to have a simple expression wrapped in a closure. With a
little bit more work tonight, I thinned down the syntax even further, for
cases where the closure is going to be a single expression:

    var f = ->(i) i * i;

Where the sequence after the "`->`" and the parameter list could be a single
expression instead of being a block, and the result of that expression would
be implicitly returned when the closure was executed. For reference, the same
thing in NQP-rx would be:

    my &f := -> $i { $i * $i };

So we're doing pretty good, for these simple cases.

With this newest version of the syntax, the Query example can be reduced to:

    int sum = as_queryable(data)
        .map(->(i) i * i)
        .filter(->(j) j % 2)
        .fold(->(s, i) s + i)
        .data();

This is just hypothetical, of course. I have written this patch and tested it
lightly. It does have some issues, especially with built-in functions and a
few other situations where the expression parser gets confused. Also, whether
the patch works or not, there's no expectation that NotFound would accept it
into Winxed master. I like it, and I would make immediate use of it, but I can
get along just fine without it too.

I'm going to keep playing with my patch and see what else I can do with it. I
need to fix some of the brokenness that I mentioned, and play around with the
implementation a little bit. It's easy to play with since the Winxed source
is so hackable.  I'm also looking at creating a new syntax for easily looking
up functions in namespace. Right in winxed, if we wanted to call a function in
a different namespace we would have to do:

    using Foo.bar;
    bar();

For comparison, NQP-rx would allow us to write this in one line:

    Foo::bar();

I would like something similar for Winxed, something that would allow us to
call a function from another namespace directly with an implicit lookup,
without needing a separate, explicit lookup instruction. I don't know what
this syntax would be. I was thinking we could reuse the `using` syntax like
this:

    (using Foo.bar)();

But that seems kind of crappy. Maybe we could use something like this:

    *Foo.bar();

Winxed does draw inspiration from C++, and the `*` is a "lookup" in some
sense of the word. Again, this is a situation where even if I do write up a
patch, it would be up to NotFound to accept it or not. I may be better off
waiting for him to make a decision and then I can spend the effort to try and
implement it. It's not super-high priority, but it would help to make some
ugly code much more readable, and I'm all about readability.
