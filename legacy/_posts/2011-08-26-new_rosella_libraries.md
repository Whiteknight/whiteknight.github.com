---
layout: post
categories: [Parrot, Rosella, Template]
title: New Rosella Libraries
---

The other day I quietly bumped the Rosella Template library to stable status.
I've been living with the API for a while and am happy with it for the most
part. I've also gotten documentation and unit tests up to a nice level and I
felt pretty comfortable that it wasn't a buggy, incomplete piece of crap. I
don't want to claim that it's perfect, but I'm pretty happy with it as a first
attempt. There are a few internal bits that are awkward and difficult to test,
but the overall public-facing form of the library is working well enough.

It's extremely easy to write up things like unit tests and documentation when
I have a templating tool that can produce files for those kinds of things
semi-automatically.

I've talked a lot about the templating library in two [previous][template_1]
[posts][template_2], and am planning a separate post for it later this week,
so I won't go into details about it here as well. As I start putting together
more tools that use it, I'll show those off so the reader can see the new
library in action.

[template_1]: /2011/07/06/rosella_template_beta.html
[template_2]: /2011/07/28/rosella_tools_and_templates.html

Instead, today I want to talk about some of the new Rosella libraries I have
been playing around with. Some or all of these might become a stable part of
Rosella some day too, but right now they aren't quite ready for prime time.

### Rosella.Assert

Rosella.Assert, formerly known as "Contract" is a library for debugging. It
provides a few interesting tools: Runtime assertions, debug logging, and
contracts. All of these features read a global flag to determine if they are
on or off. If off, calls to the various assert, debug, and contract routines
all do nothing. The calls themselves aren't completely removed, but they do
short-circuit and exit early with no side-effects. Several people, especially
GSoC students have asked for these kinds of debugging routines, so it's about
time somebody added them.

The library basically does nothing unless you turn it on, so it won't
interfere with your code at all unless you enable it. To turn on the Assert
library, you do this:

    var(Rosella.Assert.set_active)(true);

Without that line of code, most calls in the Assert library do nothing at all.
The calls are still made, however, but they check the flag and immediately
return if it's not activated. Winxed does do dead code elimination from
conditionals with constant expressions. That dead-code elimination along with
a new `__DEBUG__` constant NotFound added recently, means that you can make
assertions disappear entirely if you want:

    if (__DEBUG__)
        var(Rosella.Assert.debug)("This message probably won't appear");

The Assert library provides assertions too. So you can start peppering your
code with calls to the `assert` function:

    using Rosella.Assert.assert;
    assert(1 == 1);

Notice that the conditional here is executed before the `assert` function is
called, so side-effects aren't invisible. However, a different form of
assertion takes a predicate Sub, which won't be evaluated at all if the
library is turned off:

    using Rosella.Assert.assert_func;
    assert_func(function() { return 1 == 1; });

It's not very pretty, but it does what you expect it to do. If the Assert
library is enabled *and* the assertion condition fails, an error message and
backtrace are printed. Otherwise, nothing happens. Likewise, you can make the
code disappear entirely using the `__DEBUG__` flag.

The new library also provides contracts in two flavors: Object interface
contracts, and function contracts. In the first, we verify certain features
of an object: Does it have the necessary list of methods? Does it have the
necessary attributes? We can use the contract to verify that the object has
the expected interface, or throw an assertion failure if not. In the second
type, we can insert predicates into an existing function or method, to do
pre- and post-call testing of values. For instance, to assert that the first
argument of a call to method `bar()` on class `Foo` is never `null`, we can
set up this assertion:

    var c = new Rosella.Assert.MethodContract(class Foo);
    c.arg_not_null("bar", 0);

likewise, to verify that the method `bar()` never returns a null value, we can
do the following:

    c.return_not_null("bar", 0);

That will automatically inject predicates into the method, which will be
checked every time the method is invoked, if the Assert library is enabled.
If a check fails, you get an exception and a backtrace. If you turn off the
library, no checking happens and method calls happen like normal with no
interference or slowdown.

One last detail, the Assert library integrates with the Test library, if you
have them both loaded together. If you use Assert in a test suite using
Rosella Test, you can use assertions and contracts as tests directly, and
things printed out with `debug()` are printed out as normal TAP diagnostics.
It's quite handy indeed, because if you have put the assertions and contracts
into your code, you now have test code running *inside* your program, testing
things that would otherwise be hard to get to. Setting up predicate assertions
on methods in your code is a handy and useful replacement for mock objects, if
mocks aren't your kind of thing.

This library has a lot of potential to be used as a debugging and testing
aide, and I expect to be using it a lot in my own work once I get some of the
last details sorted out.

### Rosella.Reflect

Rosella.Reflect is a library for doing easy, familiar reflection. Basically,
it's an abstraction over methods and tools already provided by Parrot's
built-in types, but with a nicer interface. It is the same motivation as I
had for the FileSystem library, which is like a much nicer veneer over the
OS PMC and a handful of other lower-level file-manipulation details provided
by Parrot. Right now Reflect is an early-stage plaything, but it's already
looking nice and I have plenty more things to add to it.

With Rosella.Reflect, you can do things like this:

    var f = new Foo();
    var c = new Rosella.Reflect.Class(class Foo);
    var b = c.get_attribute("bar");
    b.set_value(f, "hello");        # Set Foo.bar = "hello"
    var m = c.get_method("baz");
    m.invoke(f, 1, 2, 3)            # f.baz(1, 2, 3)
    var x = new Bar();
    m.invoke(x, 1, 2, 3)            # Error, x is not a Foo

Basically, it provides type-safe object wrappers for classes, attributes and
methods. It provides routines for iterating attributes and methods, dealing
with the indirectly, and doing other reflection-based tasks too.

In the future I want to add tools for building classes at runtime, tools for
exploring packfiles, namespaces, and lexicals, and doing a few other things.
This library is pretty heavily influenced by some of the things Parrot user
Eclesia has been doing with his Eria project. It gives a nice object-based
way to interact with some things in Parrot that don't always have the most
friendly interfaces.

This library is very very young and is mostly a prototype. I am looking for
more things to add to it, and hope it will become more generally useful. It's
complicated by the fact that the upcoming 6model refactors could radically
change the way we do some types of reflection, so I don't want to reinvent
any wheels.

### Rosella.Dumper

Rosella.Dumper is a replacement for the Data::Dumper library that ships with
Parrot. It uses an OO interface with pluggable visitors and configurable
settings. It's very early in development but it's already much more
functional and usable than Data::Dumper. With the new interface, you can do
something like this:

    var dumper = new Rosella.Dumper();
    string txt = dumper.dump(obj);
    say(txt);

The Dumper object contains several collections of DumpHandler objects, which
are responsible for dumping out particular types of object. DumpHandlers are
arranged into 4 groups: type-based dumpers that dump objects of specific
types, role-based dumpers which dump objects that implement a given role,
miscellaneous dumpers which are given the opportunity to dump anything else,
and special dumpers for things like null and anything that falls through the
cracks. By mixing and matching the kinds of things you want to see dumped,
you can customize behavior. By subclassing the various bits, you can change
behavior and output formatting.

This library is pretty straight forward, and is already pretty generally
useful. I have a few decisions left to make about the API and some of the
default settings, but it's useful and usable now and I've already employed it
myself for several recent debugging tasks.

### Rosella.CommandLine

This is the newest prototype library of all. So new that as of the publishing
of this blog post I haven't pushed the code to github yet. Rosella.CommandLine
is a library for working with the command line and command line arguments in
particular. Basically, it's a replacement for the GetOpt::Obj library which
comes with Parrot, along with a few other features for making program entry
easier. To give a short example, here's a test program I've been playing with
using the new library:

    function main[main](var args) {
        var rosella = load_packfile("rosella/core.pbc");
        var(Rosella.initialize_rosella)("commandline");
        var program = new Rosella.CommandLine.Program(args);
        program.run(real_main);
    }

    function real_main(var opts, var args, var other) {
        ...
    }

The `main` function initializes Rosella and loads in the CommandLine
library. Then it creates a `Rosella.CommandLine.Program` object, to handle
the rest of the details. The Program object takes the program arguments and
automatically parses them out into a hash and some arrays based on
some basic syntax rules. You can specify formats like you do in GetOpt::Obj if
you want, or the library can parse them by default rules and just pass you the
results. The `run` method of the Program object takes a reference to a Sub
object to treat like the start of your program. It sets up a try/catch block
to do automatic error catching and backtrace printing. Also, it can be used to
dispatch certain arguments to different routines entirely, which is useful if
you need to set up routines for printing out help or version information:

    program.run(real_main, {
        "help" => function() { ... },
        "version => function() { ... }
    );

The argument processing is done by a new `Rosella.CommandLine.Arguments`
class, which mimics much of the behavior of GetOpt::Obj, but has a few subtle
differences, which are partly because it's early in the implementation and
partly because I like certain syntaxes better than others. Also, if you return
an integer from your `real_main` routine, that integer will be the exit code
of the process, which should be familiar for most C coders and their ilk. If
you return no value, or if you use the `exit` opcode, things will continue to
behave as you would expect. As with all Rosella libraries, there will be
plenty of opportunity for subclassing and customization, so if you need
something different from the provided defaults, it will be easy to change
things.

I have a couple other projects in mind that I want to start playing with, some
in Rosella, and some that intend to use Rosella to build bigger things. I'll
certainly share more information about whatever else I am planning in future
blog posts.
