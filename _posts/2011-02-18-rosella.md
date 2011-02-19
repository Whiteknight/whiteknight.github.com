---
layout: post
categories: [Parrot, Rosella]
title: Library of Libraries, Rosella
---

In the past few days I've created two new library projects: Parrot-Test
and [Parrot-Container][]. The goal of each of these two projects was to
provide a set of reusable libraries for people to make some common tasks
easier and enable easy use of best practices.

[Parrot-Container]: /2011/02/16/introducting_parrot-container.html

While working on Parrot-Container I realized that I wanted to use Parrot-Test
to implement the hest harness and test suite. While working on Parrot-Test I
realized that I really wanted to use some of the decoupling and new
event-based features of Parrot-Container. Also, once I started adding an
events library to Parrot-Container, I realized that project was much more than
just a dependency injection container project and really needed to be renamed
to show that it was bigger. Proper mock objects for Parrot-Test were also
going to require a proxy object library, which didn't even have a home yet.

In short I have a lot of little things I wanted to do, these two libraries
really were only small parts of my larger overall plans, and many of the
little things I am working on have lots of opportunity for code sharing.

Today I've merged those two projects together into a new project called
[Rosella][]. Rosella is a library-of-libraries project, where I aim to provide
a series of reasonably-sized building block libraries for people to use.
Rosella is not a framework per say. It's more modular than that. At the moment
it is a collection of about 5 libraries (and growing) that can be used
independently of each other. There are opportunities for interplay and code
reuse, but I am trying to keep each individual building block as small as is
usefully possible. Pick what you want, leave what you don't, and be empowered
by having these cool new tools at your disposal. Here's a quick rundown of
what Rosella offers right now:

[Rosella]: https://github.com/Whiteknight/Rosella

## Rosella/action.pbc

Actions are bundled-up commands that can be created once and executed later
on a target object. In the most simple case, `Rosella::Action::Method`, the
action is a method call on the given invocant with a pre-specified list of
parameters. Once you have the Action object all bundled up, you can pass it
around and call it any time you want.

Arguments to an Action are themselves complicated objects which provide
resolution behavior: when you ask for the appropriate value, it may return it
directly, calculate it, search for it, etc. If you're using the Action library
with the Container library (described below), there is an action argument type
that looks up the value in the container.

Actions are useful because you can package up some kind of command that you
want to execute, store it for later, and execute it whenever you want.

For people familiar with design patterns, this is essentially an
implementation of the "[Command][command_pattern]" pattern, but with a bit
more flexibility in resolving parameters.

[command_pattern]: http://en.wikipedia.org/wiki/Command_Pattern

## Rosella/container.pbc

This is the dependency injection container that I discussed in a recent post.
You register a type with the container, and later can resolve that type to
produce an instance. In essence, you tell the container ahead of time the name
of the type and specify a list of rules for how to create and initialize an
object of that type. Then when we resolve, we get an instance (either new or
cached) and execute all the inialization actions to properly initialize that
object.

The `Rosella::Container` type is an implementation of the "[Inversion of
Control][ioc_principle]" (IoC) and "[Dependency Injection][di_principle]" (DI)
principles.

[ioc_principle]: http://en.wikipedia.org/wiki/Inversion_of_Control
[di_principle]: http://en.wikipedia.org/wiki/Dependency_Injection

## Rosella/event.pbc

The new Events library allows you to create decoupled events in your
application. Early in your program you define a list of events that your
program is going to use. Then, you register handlers or listeners who are
interested in those events. When the event is raised later in the program,
all of the listener Actions are executed to be notified about the event.

This is essentially an implementation of the
"[Observer/Observable][observer_pattern]" pattern.

[observer_pattern]: http://en.wikipedia.org/wiki/Observer_Pattern

## Rosella/prototype.pbc

This is a very small library that implements a Prototype system. You register
prototype objects with the PrototypeManager. Later in your program when you
need an object, you ask the PrototypeManager to clone you a copy of the
registered prototype. Think about JavaScript's object system, and this is
essentially that. Notice that `Rosella::Container` can be used to instantiate
prototypes as well, but this library is much thinner and single-purpose than
that.

This is an implementation of the "[Prototype][prototype_pattern]" pattern.

[prototype_pattern]: http://en.wikipedia.org/wiki/Prototype_pattern

## Rosella/xunit.pbc

Borrowed in large part from Kakapo, this is a "mostly" implementation of
[xUnit-style][xunit] unit tests. You define a class which is a descendant of
`Rosella::Testcase`, create a suite from it, and then run the suite.
`Rosella::Testcase` and the related machinery automatically finds and executes
all test methods in the class and generates all the necessary output. In this
case the output is TAP, but the system is extensible enough to allow for other
output formats.

[xunit]: http://en.wikipedia.org/wiki/XUnit

## Rosella/tap_harness.pbc

This library provides all the necessary logic for implementing a complete,
functional TAP harness. By using this library, you can create a
fully-functional TAP test harness in about 10 lines of code. It has some
pretty nifty features too, which other harnesses in the Parrot ecosystem
probably cannot match.

For a bit of perspective, here is an entire test harness implementation using
`Rosella::Harness`:

    INIT { pir::load_bytecode("rosella/tap_harness.pbc"); }
    my $harness := Rosella::build(Rosella::Harness);
    $harness.add_test_dirs("NQP", 't', :recurse(1));
    $harness.run();
    my $output := Rosella::build(Rosella::Harness::Output::Console);
    $output.show_results_summary($harness);

Thats it. Six lines of code and you have a complete TAP harness for your
project. Quite powerful, considering how many projects in the Parrot ecosystem
have TAP-based test suites.

## Future Libraries

There are a few other libraries I am thinking about as well. Here is a quick
list of things I am actively planning:

1) proxy: A proxy-building library which can create proxy objects for
   use in place of other objects for a variety of purposes.
2) mockobject: By using a proxy object to intercept method and vtable calls
   we can add test logic to create mock object-based testing tools.
3) test: A test library, similar in interface to Perl5's Test::More. All your
   favorite functions will be there (ok, nok, is, isnt, diag, etc). The
   difference is that the output will be configurable just like the xunit test
   library, and the two will be compatible. You'll be able to output test
   results in multiple formats, including but not limited to TAP.

I have a few more ideas in my head, but none as concrete or as thought-out as
these. I'll certainly be making more posts as work progresses on these tools.

So that's a quick introduction to my new Rosella library. I'm sure I'll be
writing more about it in the days and weeks to come.
