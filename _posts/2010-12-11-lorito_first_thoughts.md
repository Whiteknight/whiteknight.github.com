---
layout: post
categories: [Parrot, ParrotDesign, Lorito]
title: Lorito Design, First Thoughts
---

Thursday night was the big face-to-face meeting between Christoph, Allison and
chromatic to discuss the future of Parrot, especially as it pertains to
*Lorito*. Lorito has traditionally been many things depending on who you
talked to or how you thought about it about it. Now it's starting to take a
more concrete form and it's not really what a lot of people thought it was
going to be.

For a very long time now I've been arguing that PIR is a problem for Parrot.
It's a lousy assembly language and it's relatively high-level of syntax and
semantics means it is difficult to use it for things that we really want to
do: JIT, Concurrency, Security, Extensibility. My argument was that we needed
to dramatically simplify the set of opcodes that Parrot worked with to help
improve performance in a general sense but also to enable technologies like
those I listed above. The original name for this reduced opcode set, or
"microcode set" was L1. Eventually, for a variety of aesthetic and poetic
reasons, L1 was rebranded as Lorito.

I was looking at Lorito from the context of my background with microprocessor
hardware. When you simplify the opset of a processor you gain a number of
benefits. Simple, fixed-format instructions are much easier to decode, so your
decoding and control logic becomes more simple. A smaller opset can lead to
less total logic, a smaller hardware footprint, and eventually to higher clock
speeds. This is all not to mention the fact that tooling for a simple opset
is dramatically simplified: The effort required to write your own MIPS
assembler is orders of magnitude less than the effort required to write a
fully-functional x86 assembler, for instance. Disassemblers and debuggers as
well become trivially easy to write.

In some modern CISC processor architectures, the processor "front-end"
translates the incoming CISC ops into a sequence of internal RISC "micro-ops".
These micro-ops, though there are more of them, are easier to work with and
are passed into an internal "microcore" which is highly optimized and operates
at a higher clock speed. This is exactly the metaphore that I was using for
the relationship between PIR and Lorito: PIR "meta-ops" are translated to
a sequence of simpler and faster Lorito ops for actual execution.

There are many differences between a hardware processor implementation and a
software virtual machine implementation, of course. Parrot can't just bump
up it's internal clock speed or make the underlying hardware any faster. What
it can do, however, is make use of the decreased complexity to help simplify
and optimize it's algorithms. Also we should be able to enable new features.
Everybody loves features.

Of course, once you start talking about making radical changes to one aspect
of Parrot to help make the addition of new technologies more straight-forward,
it is kind of hard not to talk about fixing other things that represent either
long-time technological problems or which look to become bottlenecks in the
future. Looking at some of the raw meeting notes generated from the meeting
last night, it seems like the trio definitely took a no-holds barred
approach to the Lorito design. In short, Lorito is more than just a new
dialect of microcode. It's going to represent a major architectural shift for
parrot that may be jarring but will ultimately help us to push forward the
boundaries of what we have been capable of in the past and what all dynamic
runtimes may be capable of in the future.

The most immediate change that I see is the idea that we no longer have a
central interpreter object, but instead run everything off the context object.
This mirrors closely an idea that Chandon and I were talking about during his
GSoC threading project: A single global interpreter causes a lot of problems
for any concurrency implementation because we are frequently going to be
sharing--and therefore contending and conflicting--the important bits of
global data. His solution, which I very much liked, was to break the
interpreter into two parts: A small global part which contains the things that
absolutely need to be shared, and a local part which contains things that can
be independent on individual threads. In this way we can avoid having a huge
"Global Interpreter Lock" (GIL) to lock the interpreter as a whole and can
start worrying about sharing only a small number of global items on a per-case
basis.

It's looking like the Lorito meeting wants to pursue a similiar goal. Instead
of breaking up the interpreter into a global- and local- version, we call the
local version the "context" and attempt to shrink the global- version down to
nothingness. There are many benefits to this approach, but many questions
arise from it as well. For instance, how do we manage dynamic class
definitions across threads? Does each thread need a local copy of a metaclass
object, or is there a central reference point? If the former, how do we handle
cases where local copies of a metaclass definition are out of sync with each
other? This can lead to cases where an object of the "same" class on two
threads in a single application have radically different semantics. In the
later case, we need to tightly synchronize lookups on the global metaclass
definition, which adds a non-negligible performance drain on every single
cross-thread operation that uses them: object creation, method lookup and
dispatch, etc.

This isn't really a problem for Lorito, any virtual machine runtime that
supports a reasonable object model is going to have to contend with issues
like this. In a multiple-cores and multiple threads kind of world, the only
things that we don't have to worry about sharing are things we explicitly keep
as immutable. Strings are certainly immutable, and I would make the strong
case that bytecode be immutable too, at least after it has been loaded into
the interpreter for execution. Maybe to get around the issue above we enable
a mode where any PMC can be made read-only, and we require that only read-only
PMCs can be shared. Or, we define copy-on-write (COW) semantics for PMCs that
say we can share a read-only copy, but if somebody wants to make a
modification to one we do a full copy and split the two apart never to be
reattached (or, not without explicitly sending a message to do so).

I don't want to harp on and on about concurrency, but I see that as being both
a huge necessity for Parrot in the long term and a huge challenge that we need
to start planning for *right now*. If we make decisions today that are not
concurrency-friendly, we are going to be regretting them in the future when
the time comes to actually start implementing a concurrency system. Of course,
like I mentioned above, no matter what we do there are going to be some
problems, and in many cases our only option will be to pick which problems
we want to deal with and then do our best to minimize them.

I think that the new system will work well with a basic [green threads][]
implementation for now, so maybe that's something I try to put together on
top of Lorito before we have to worry about a more complete threading system.

[green threads]: /2010/06/02/parrottheory_green_threads.html

Some other things that I like from the initial Lorito notes are that STRING
and PMC types are going to be combined together, so the user will only have
to worry about String PMCs (with proper methods, etc) instead of having a
separate STRING structure. This is going to be a big benefit in many places,
especially inside our GC where it will reduce complexity by a significant
amount. I also like the idea that the MOP will be more deeply integrated into
things, and that we handle security at the MOP level instead of at the op
level. That's a great idea and I think it solves a lot of conceptual problems
Parrot will be running into in the future.

I think Christoph mentioned that he was going to [write a post][reparrot]
about Lorito sometime soon once he has time to get all his thoughts together.
I'm looking forward to that post of his, and I'm sure I'll have more reactions
to post about it when we get there.

[reparrot]: http://reparrot.blogspot.com/
