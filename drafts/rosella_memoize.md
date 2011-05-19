---
layout: post
categories: [Parrot, Rosella, Memoize]
title: Rosella Memoize Library
---

Two days ago I upgraded the [Rosella Memoize library][memoize_page] to
"stable" status. This followed a brief but concerted period of work on that
library. What does it mean to be "stable" in the context of Rosella? What it
means in my head, at the most basic level, is that the library provides an API
which is usable and which I am mostly happy with. The word "usable" implies
that it be usable by people besides myself, which means the code must be
documented and it must be well-tested. I don't claim that a stable Rosella
library is perfect, or that it is free from bugs, or that I won't be making
random changes to it over time as new ideas come to me. What I do claim is
that it is ready for other people to start looking at and playing with. The
future course of the library will be based, at least in part, on feedback I
receive from users.

[memoize_page]: http://whiteknight.github.com/Rosella/libraries/memoize.html

The Memoize library provides memoization. [Memoization][memoization], for
people not familiar with the term, is a form of caching for subroutines. When
a function is [pure][function_purity], and the cost of repeatedly calculating
a result is more expensive than a lookup in a cache, memoization could bring
a nice performance win.

[memoization]: http://en.wikipedia.org/wiki/Memoization
[function_purity]: http://en.wikipedia.org/wiki/Pure_function

We take a function that we want to memoize and pass it to the Memoize library.
We get back a new function called a "memoizer" which, when invoked, attempts
to look up the result in a cache first. If the cache has the value, we return
it directly and save on extra computation time. If not, the memoizer calls the
original function and adds the result to the cache.

The Rosella Memoize library provides a few mechanisms for memoization. There
are **simple memoizers**, which are basically closures with cache logic
surrounding a call to the original function, and there are **proxy-based**
memoizers which use the Rosella Proxy library to intercept VTABLE_invoke and
perform cache logic there. The difference between these two approaches is that
simple memoizers have lower overhead and better performance, but offer lower
flexibility than the proxy-based alternatives. For instance, we can query a
Sub to determine if it is a proxy or not (and if not, we can intelligently
decide to memoize lazily). Also, we can get access to the original Sub object
and the cache object to play with. The simple memoizer cannot do any of these
things.

Here's a quick example, which comes almost directly from the Rosella test
suite. First, the case of simple memoizers:

    sub my_func($a) {
        pir::say("Arg: $a\n");
        return $a + 5;
    }
    my &memoize := Rosella::Memoize::memoize(my_func);
    pir::say("Result: " ~ &memoize(5));
    pir::say("Result: " ~ &memoize(5));
    pir::say("Result: " ~ &memoize(5));
    pir::say("Result: " ~ &memoize(5));

This, of course, prints the following output:

    Arg: 5
    Result: 10
    Result: 10
    Result: 10
    Result: 10

We only call the `my_func` function once. Every subsequent access looks up the
return value in the cache and avoids the function call. Of course, we also
lose out on the side-effect of printing a message to the console. The Memoize
library cannot currently account for console output, although it's something I
might think about adding if users are keen on the idea. I suspect it wouldn't
be worth the costs and limitations, however, since there is no way I could
automatically intercept communications to all open filehandles.

Now here's a similar example, but using proxy-based memoizers instead:

    sub my_func($a) {
        pir::say("Arg: $a\n");
        return $a + 5;
    }
    my &memoize := Rosella::Memoize::memoize_proxy(my_func);
    pir::say("Result: " ~ &memoize(5));
    pir::say("Result: " ~ &memoize(5));
    my $cache := Rosella::Memoize::proxy_cache(&memoize);
    $cache.get_item(5).update_value(20);
    pir::say("Result: " ~ &memoize(5));
    my &orig_func := Rosella::Memoize::proxy_function(&memoize);
    pir::say("Result: " ~ &orig_func(5));
    pir::say("Result: " ~ &memoize(5));

This gives us more interesting output:

    Arg: 5
    Result: 10
    Result: 10
    Result: 20
    Arg: 5
    Result: 5
    Result: 10

By getting access to the Cache object, we can pre-fill the cache with values
known at runtime (and only ever calculate values which are sufficiently
"rare"), or we can update the values in the cache later to account for errors,
or whatever. Also, we can get access to the original function if we absolutely
need access to it and want to bypass the cache. For instance, we can get the
original function and memoize it again, to have a separate cache with
different properties. By accessing the cache directly, we end up with
something like a lazy lookup table, where we can insert values we know and
automatically compute and cache values we don't know. I don't want to say the
possibilities are "endless", because this is still a pretty focused and
limited tool. However, there is power here, and it can be extremely useful for
cases where it's needed.

One place where proxies are absolutely required is for in-place method
memoization. The Memoize library allows you to inject a memoize proxy directly
into an existing class in place of an existing method. Memoization appears
magically for common object types, and is completely transparent for most
users. Later, you can transparently unmemoize the method call as well without
any ill effects.

    Rosella::Memoize::memoize_method("String", "to_int");
    Rosella::Memoize::unmemoize_method("String", "to_int");

Here's one more example, lifted directly from the test suite, of a variant of
the Y-combinator with built-in memoization provided by the library:

    my $answer := (Rosella::Memoize::Y(sub($g) {
        return sub($n) {
            if ($n <= 1) { return $n; }
            return $g(+$n - 1) + $g(+$n - 2);
        };
    }))(50);

The answer, for those trying to do the math in their heads, is
[12586269025][fibonacci_50]. With memoization, the library computes this value
in a fraction of a second. Without memoization, this calculation can strain
a modern computer for several seconds and even minutes. When I went to test
this myself just now, the version with memoization returns the correct value
in 0.0002 seconds. I had to kill the unmemoized version after about 2 minutes.

[fibonacci_50]: http://planetmath.org/encyclopedia/ListOfFibonacciNumbers.html

I have not hit the end of the road with this library. The default caching
mechanism leaves a lot to be desired, and the library does not currently
support fancy returns like named return values or multiple return values.
These are things that will likely make it into later revisions but which just
aren't there yet. There are a handful of other nits as well, that I am
planning to work on, later when my schedule clears up.
