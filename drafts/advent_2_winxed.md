---
layout: post
categories: [Parrot, Advent2011]
title: Advent 3 - Winxed
---

For the third post in my almost-advent calendar, I'm going to talk about
Winxed.

Winxed, written by long-time Parrot hacker NotFound, is quite an interesting
project and is definitely worth learning more about. It serves as something
of a counterpoint to NQP, the other lower-level Parrot language in the ecosystem
but the two aren't competing. I think they are very complimentary. Where NQP
is designed to help building compilers (Rakudo Perl6 especially), Winxed seems
much more geared towards writing libraries and utilities. It's for this reason
that I've written my library project, Rosella, in Winxed.

Winxed is written in itself using a home-brewed recursive descent parser. To
perform the bootstrapping, the stage 0 winxed compiler is written in C++.
In stage 0, a paired-down version of the language is used to compile the stage 1
compiler which is written in that subset. Stage 1 is used to compile stage 2,
which is a more full-featured version of the language. Stage 2 is used to
compile itself into stage 3. Stage 3 is what you and I use when we install
Winxed.

It sounds much more complicated than it is, but the net result is clear and
simple: Winxed written in Winxed. If you know Winxed, or are familiar with
some of the big languages it is inspired by (C++, Java, C#, JavaScript),
you'll be able to not only write software for Parrot, but also be able to
hack on the Winxed compiler itself. I've submitted a few patches and feature
additions, and I've found it to be a pleasure to work on.

It's not anything goes on the compiler, however. NotFound maintains pretty tight
editorial control over the software. The consistency of vision and planning
does produce refreshingly nice results, though.

Since bundling with Parrot in th 3.6.0 release, which incidentally is when
NotFound started keeping track of version numbers, the language has come a
long way: Various optimizations, a debugging mode with optional asserts
and conditionals, several new built-in functions, support for multiple
dispatch and most recently a new "inline" feature which allows you to inline
certain types of code for performance improvements by avoiding extra PCC
calls.

Here is a random-ish sample of Winxed code that I've been playing around today
which is written in Winxed for testing some new ideas I'm thinking of adding to
Rosella:

    function main[main](var args)
    {
        var rand = Rosella.Random.default_uniform_random();
        var mutator = new Rosella.Genetic.Mutator(
            function() {
                float value = float(rand.get_float()) * 1000.0;
                return new Data(value);
            }
        );
        var e = new Rosella.Genetic.Engine([10, 3, 0], mutator,
            function(var d) {
                int x = int(d.value) - 200;
                ${ abs x };
                return x;
            }
        );
        var w = e.run(500).first().data();
        say(float(w.value));
    }

This sample is a driver program for a new Genetic Algorithms library that I'm
playing with. This toy example uses the Genetic Engine type to pick some random
numbers at each generation for five hundred generations to try and get a random
number which is closest to `200.0`. I'll write more about that library in the
future if anything ever comes of it, but that's not the important part of this
example. The important part is the Winxed syntax. The syntax should be
immediately familiar to anybody who has used JavaScript or C++ and any of its
descendents. Here we can clearly see closures created with the `function`
keyword, creating objects with `new` (Winxed is OO, not prototype-based like
JavaScript), low-level types like `float` and `int`, Parrot Sub flags like
`[main]` (AKA `:main` in PIR) and calling PIR opcodes directly (the `${ ... }`
syntax). Winxed in a nutshell is an an answer to this question: What would it
look like if I took a language like C++ or JavaScript and bent it to work
closely with Parrot?

I don't know where NotFound is planning to take Winxed in 2012. His relentless
pursuit of new features, better internals, more optimizations, better
diagnostics and other features leaves open many potential avenues for him to
travel down next year.

I have a few ideas for features I might like to propose and provide patches,
many of which will be internal changes to mirror changes that will need to be
made in Parrot. I have been thinking about submitting a patch to add some
cleaner syntax for parameter and argument flags instead of using PIR flags by
name directly. I'm also keen to get Winxed updated to use some of my new PCC
changes as soon as they are available. It will make both a great test case for
the new functionality and a good demonstration of any performance benefits to be
had. Eventually I would like to get Winxed updated to generate .pbc packfiles
directly instead of generating PIR and using IMCC as an intermediary. Again,
this would make an excellent demonstration of the functionality, when we have
it ready to test.

I personally use Winxed to implement my Rosella project and the first stage of
my JavaScript compiler "Jaesop". Other people are using it as well. Hacker
plobsing has used it for a few projects, including an OMeta parser port. Hacker
benabik is using it for PACT, a re-design of the Parrot Compiler Toolkit.
Dukeleto is writing bindings for libgit2 in Winxed. NotFound is writing xlib
bindings in Winxed called Guitor (and some of the example programs are actually
quite impressive already). There are many other examples of projects that are
or will be written in Winxed, and I'm sure the number will only increase in
2012.

If you're doing systems-level library or utility work on Parrot in 2012, Winxed
is probably the language you are going to be using.
