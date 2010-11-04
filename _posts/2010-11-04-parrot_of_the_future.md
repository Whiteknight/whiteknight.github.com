---
layout: post
categories: [Parrot]
---

Several days ago I wrote up
[a blog post](http://whiteknight.github.com/2010/09/19/parrot_as_a_mature_platform.html)
talking about a vision for the development of Parrot moving forward. I asked a
simple question: What does Parrot need to provide in order to be considered a
compelling choice for compiler developers at the time? After some comments,
I posted a [followup](http://whiteknight.github.com/2010/09/20/woe_is_parrot.html)
too, expounding some of the same points. Today, after much waiting, it's time
to continue that train of thought and put down more vision about what Parrot
can and should become in the next few months and years.

There are several things happening in Parrot core development *right now* that
we should be getting more excited about:

Christoph Otto,
[newly appointed architect](http://reparrot.blogspot.com/2010/10/parrot-has-new-architect-what-now.html)
has been pushing forward with design of the next-generation core, called
Lorito. I've talked about Lorito (formerly "L1") several times on this blog,
so I won't go into detail about it right now. Lorito itself isn't flashy or
useful in it's own right, but it should serve as an enabler for many cool new
things like a massively improved JIT engine, among other things.

Jonathan Leto, despite his busy schedule, has been pushing forward with the
migration of Parrot's source code repository to Git. Again, this isn't going
to improve Parrot directly, but I hope it will be an enabler for getting more
people involved in Parrot and getting our code more exposure. I expect the
migration to be completed sometime this week, and then we will see how things
turn out.

Vasily Chekalkin and chromatic have been pushing forward with Parrot's new
garbage collector, which isn't ready quite yet but is already showing some
major performance potential. Once the new GC lands and is properly tuned, I
don't think we will be able to blame the GC for our performance woes anymore.

I've been working on revamping our sorely-neglected embedding API. Some common
tasks that you would expect to be able to do with an embeddable interpreter
are either extremely difficult or absolutely impossible. Plus, we don't have
enough API functions, so external projects are breaking encapsulation left and
right to perform even basic tasks. A new API should make Parrot easier to use
and should serve as a motivator to users who want to employ Parrot for various
projects.

These are all good things, but they certainly aren't everything Parrot is
going to need in the short-term, much less the long term. Let me put forward
a hypothetical timeline for the next few supported releases on how Parrot
could realistically develop into a world-class dynamic language runtime:

__2.9__: Out already.

__3.0__: Parrot has a new GC. There are some kinks, and some tuning needed
(especially on "exotic" platforms), but performance and memory consumption
have improved. The first stage of a new embedding API is in place, and several
projects are starting to use it to great effect. New NCI system is merged in,
which gives much more flexibility for wrapping and using existing native
libraries.

__3.3__: Initial designs for Lorito are out, and a prototype is in place. We
can start putting together some tools to work with the new language, including
a basic assembler, and a compiler to convert Lorito into C. With this, we can
start rewriting parts of Parrot in Lorito on an experimental basis. Parrot has
an implementation of green threads, and most IO operations are non-blocking.
This brings a performance boost for properly-written IO-heavy programs.

__3.6__: Lorito is maturing, and all PIR ops have been rewritten in terms of
Lorito ops. New designs are available to change L-value semantics, and the new
semantics are to be implemented directly in Lorito. The second stage of the
new embedding API in in the works, which is necessitating some changes to the
exceptions system, among others, to play more nicely with embedding
applications.

__3.9__: New L-value semantics are in place, and NQP (and other compilers)
start to see significant performance improvements. With Lorito more mature,
and more heavily used in Parrot, we start designing a new JIT system. We also
start designing a new object metamodel system, since we're about to rewrite
the old one anyway. A new context-threaded runcore increases performance
of interpreted Lorito by a reasonable amount.

__4.0__: The new object metamodel is in place. Combined with other Lorito
improvements, this brings a significant performance improvement for most
method invocations, and provides a much more flexible substrate for HLL
metamodels. An AOT compiler is implemented, mostly as an experiment
in understanding the compilation of Lorito opcodes using LLVM. A prototype of
the JIT is in the works.

__5.0__: A new JIT is in development that uses tracing to decrease overhead
and modestly increase performance. Parrot is natively embedded in, or can be
added through plugins to, several well-known applications.

__6.0__: A new concurrency system is being designed that integrates with CPS
and brings parallelism to Parrot without many of the shortcomings of other
low-level threading implementations. A handful of high-profile compiler
projects have been, or are being, implemented on Parrot. At least one Linux
distribution ships a major language compiler which is built on top of Parrot.

This isn't too bad a road-map for the next three years of Parrot. Some
portions of it are aggressive, but some parts of it are extremely reasonable.
Plus, this is just a high-level overview, improvements are going to be made to
other Parrot subsystems than those I mention above, bringing new capabilities
and various other improvements and refinements. We should all be excited about
the coming years and months, because they promise to bring some great things
to Parrot.

What questions or suggestions do people have about this timeline? Are there
things anybody would like to see changed or added?
