---
layout: post
categories: [Parrot, Rosella]
title: Library of Libraries, Rosella
---

In the past few days I've created two new library projects: Parrot-Test and
Parrot-Container. The goal of each of these two projects was to provide a set
of reusable libraries for people to make some common tasks easier and enable
some best practices.

While working on Parrot-Container I reallized that I wanted to use Parrot-Test
to implement the hest harness and test suite. While working on Parrot-Test I
realized that I really wanted to use some of the decoupling and new
event-based features of Parrot-Container. Also, once I started adding an
events library to Parrot-Container, I realized that project was much more than
just a dependency injection container project and really needed to be renamed
to show that it was bigger.

Today I've merged those two projects together into a new project called
Rosella. Rosella is a library-of-libraries project, where I am to provide a
series of reasonably-sized building block libraries for people to use. Rosella
is not a framework per say. It's more modular than that. At the moment it's
a collection of about 5 libraries (and growing) that are rarely
interconnected. Pick what you want, leave what you don't, and be empowered to
have these cool new tools at your disposal. Here's a quick rundown of what
Rosella offers right now:

## Rosella/action.pbc

Actions are bundled-up commands that can be created once and executed later.
An action typically combines an invokable with some data. In the most simple
case, `Rosella::Action::Method`, the action is a method call with a list of
parameters. Once you have the action bundled, you can call `.execute()` with
a target object to have the specified method called on that object with the
specified parameters.

Arguments to an Action are themselves complicated objects which provide
resolution behavior: when you ask for the appropriate value, it may return it
directly, calculate it, search for it, etc. If you're using the Action library
with the Container library (described below), there is an action argument type
that looks up the value in the container.

Actions are useful because you can package up some kind of command that you
want to execute, store it for later, and execute it whenever you want.

For people familiar with design patterns, this is essentially an
implementation of the "Command" pattern, but with a bit more flexibility in
resolving parameters.

## Rosella/container.pbc

This is the dependency injection container that I discussed in a recent post.
You register a type with the container, and later can resolve that type to
produce an instance of that type. In essence, you tell the container ahead of
time the name of the type and specify a list of rules for how to create and
initialize an object of that type. Then when we resolve, we get an instance
(either new or cached) and execute all the inialization actions to properly
initialize that object

## Rosella/event.pbc

The new Events library allows you to create decoupled events in your
application. Early in your program you define a list of events that your
program is going to use. Then, you register handlers or listeners who are
interested in those events. When the event is raised later in the program,
all of the listener Actions are executed to be notified about the event.

This is essentially an implementation of the Observer/Observable pattern.

## Rosella/prototype.pbc

This is a very small library that implements a Prototype system. You register
prototype objects with the PrototypeManager. Later in your program when you
need an object, you ask the PrototypeManager to clone you a copy of the
registered prototype. Think about JavaScript's object system, and this is
essentially that. Notice that `Rosella::Container` can be used to instantiate
prototypes as well, but this library is much thinner and single-purpose than
that.

## Rosella/xunit.pbc

Borrowed in large part from Kakapo, this is an implementation of xUnit-based
unit tests. You define a class which is a descendant of `Rosella::Testcase`,
create a suite from it, and then run the suite. `Rosella::Testcase` and the
related machinery automatically finds and executes all test methods in the
class and generates all the necessary output. In this case the output is TAP,
but the system is extensible enough to allow for other output formats.

## Rosella/tap_harness.pbc

This library provides all the necessary logic for implementing a complete,
functional TAP harness. By using this library, you can create a
fully-functional TAP test harness in about 10 lines of code. It has some
pretty nifty features too, which other harnesses in the Parrot ecosystem
probably cannot match.

## Future Libraries

There are a few other libraries I am thinking about as well. Here is a quick
list of things I am actively planning:

1) proxy: A proxy-building library which can create proxy objects for
   use in place of other objects for a variety of purposes
2) mockobject: By using a proxy object to intercept method and vtable calls
   we can add test logic to create mock object-based testing tools.
3) test: A test library, similar in interface to Perl5's Test::More. All your
   favorite functions will be there (ok, nok, is, isnt, diag, etc). The
   difference s that the output will be configurable just like the xunit test
   library, and the two will be compatible. You'll be able to output test
   results in multiple formats, including but not limited to TAP.

I have a few more ideas in my head, but none as concrete or as thought-out as
these. I'll certainly be making more posts as work progresses on these tools.
