---
layout: post
categories: [Parrot, Exceptions]
title: Exceptions Backtrace Improvements
---

A few days ago I started a new branch to work on exception backtraces. The
backtrace tells you where the program execution was at the time an exception
is thrown; extremely valuable information if you are debugging. The bug that
I was working on in particular had to do with the `rethrow` opcode, which
takes an exception which has already been thrown, modifies some of it's data,
and throws it again. Until recently, the backtrace of a rethrown exception
told you where the rethrow happened, not where the original throw did.

Here's a short example program:

    .sub main :main
        push_eh _handler
        'foo'()
        say "not caught"
        .return()
      _handler:
        .get_results($P0)
        say "caught in main"
        pop_eh
        rethrow $P0
    .end

    .sub 'foo'
        push_eh _handler
        'bar'()
        say "not caught"
        .return()
      _handler:
        .get_results($P0)
        say "caught in foo"
        pop_eh
        rethrow $P0
    .end

    .sub 'bar'
        $P1 = new ['Exception']
        $P1["message"] = "This is an exception"
        throw $P1
        say "not caught"
        .return()
    .end

In Parrot master, you get the following backtrace printed out to stderr:

    caught in foo
    caught in main
    This is an exception
    current instr.: 'main' pc 22 (test.pir:10)

In my branch, you see this:

    caught in foo
    caught in main
    This is an exception
    current instr.: 'main' pc 22 (test.pir:10)

    thrown from:
    called from Sub 'foo' pc 47 (test.pir:22)
    current instr.: 'main' pc 22 (test.pir:10)

    thrown from:
    called from Sub 'bar' pc 59 (test.pir:28)
    current instr.: 'foo' pc 47 (test.pir:22)
    called from Sub 'main' pc 9 (test.pir:3)

Seems like a victory, right? Sort of. It does satisfy the specific needs of
the original ticket, but isn't really the complete or final fix.

The Exception PMC contains a "backtrace" method which returns a backtrace
object. Actually, it  constructs and returns a complicated data structure,
made up of an array of hashes of various data. A calling program can take this
structure, iterate over it, and produce their own backtrace. Unfortunately the
new and improved backtraces that I implemented in the example above are not
represented by this structure or the 'backtrace' method. Details from before a
rethrow are not included. This is not a great thing either.

What I think we need to have instead is a dedicated backtrace object. The
backtrace object will povide a nice and programmatic interface for dealing
with bactraces and iterating over them, and will also be nestable for use in
rethrow.  If we implement things correctly, we can create these BackTrace PMCs
lazily, so we aren't adding a bunch of PMC creation overhead for common case
simple operations.

I'm starting to put together a [Tasklist] for upcoming Exceptions system
changes. There are a few big items that I want to tackle in the coming months,
like the BackTrace PMC idea that I mention above. There are several other
smaller cleanup tasks that we need to do as well. I don't plan on having some
huge monolithic refactor process. Instead, I suspect this is something we can
improve gradually over time in small steps.

[Tasklist]: http://trac.parrot.org/parrot/wiki/PackfileTasklist

We've got the big 3.0 release on Tuesday, so obviously nothing big is going to
change in master before then. After that I am going to try to merge in my
exception_backtrace changes, and a few other things as well. I know other
people have some stuff to merge also, including some things which are pretty
impressive.
