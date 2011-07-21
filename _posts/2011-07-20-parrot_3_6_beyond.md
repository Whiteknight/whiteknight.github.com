---
layout: post
categories: [Parrot, Release]
title: Parrot 3.6 and Beyond
---

Parrot 3.6 is out the door this week, thanks to release manager Jim
Keenan. It was, like so many releases before it, a mostly uneventful
release. As always, the myriad steps in the release manager guide, including
the steps which are poorly written, ommitted, or misleading, caused the usual
headaches. I won't harp on those failings here.

Any stable release is a big deal, and the X.6 release each year is no
different. It provides us with a nice median between the X.0 release and the
next one in January. In this case, it's the mid point between 3.0 and 4.0.
Now is a really great time to ask ourselves two questions: What have we
accomplished since 3.0, and what do we hope to accomplish by 4.0?

In terms of Parrot core repositor, lots has happened in the past six
months. Here's a quick rundown of some of the biggest changes:

* New Embedding API, including serious refactoring (and paving the way for
  more to come) for IMCC
* Packfile PMCs can now be used to create runable bytecode *without* needing
  IMCC. As we speak, a GSOC project is trying to add those to PCT. A new
  PackfileView PMC gives access to Packfile internals from running PIR code.
* Winxed grew by leaps and bounds, and a snapshot of it was included in the
  Parrot repo. It was recently tagged version 1.0.0.
* Information about deprecations and experimental features are included in
  machine-readable api.yaml, instead of old DEPRECATED.pod
* Generational GC, implemented and made default. Many tweaks, improvements,
  and optimizations have been added to it.
* Huge bump in test coverage for both extending and embedding APIs.

This of course is all not to mention much of the development that has occured
in related projects: NQP got 6model to help spur on the next leg of Rakudo
development, Rosella has been growing like a weed and now has several users,
Winxed is changing and improving rapidly, Several other ecosystem projects
have been springing up and growing, etc. We also got a huge bump from the GCI
program, and are knee-deep in a very productive GSoC program now too. It's
been a pretty good six months, all things considered.

So where are we heading? Well, I have a heck of a lot of stuff I want to do
personally, but not all of it is expected to get completed by 4.0. Also, other
people are working on projects with the same caveat. Here's what I reasonably
expect to happen by 4.0:

* 6model in Parrot, at least a prototype.
* More packfile-related refactors and improvements.
* Disconnect IMCC from libparrot, and offer it separately.
* Implement prototypes of new PCC system opcodes, and start migration to them.
* At least some of the first parts of the new Lorito infrastructure start
  merging into Parrot, hopefully a lot of it does.
* Start prototyping out some of the big changes for PCC.
* Improvements to our tooling, Debugger and Profiling tools especially.

There are also a few wildcards: Threading and concurrency have been something
of a hot topic recently, and I think the pressure is on for us to provide
something to support it. Exactly what we will provide is up in the air, but
I have to imagine we provide *something* by 4.0, or are well on our way
towards it. JIT, depending on how much of Lorito we can get merged, could
make an appearance as well. We could be working on a JIT compiler to compile
M0 bytecode, even if most of the system doesn't use M0 yet. The twin factors
of 6model and Lorito will, I'm sure, dramatically change the landscape in
terms of PMC types and the object metamodel. Expect to see at least some major
refactoring, rewriting, and improving of existing PMC types. Expect to see a
lot of deprecation tickets get submitted in the coming months.

Performance is a bit of a tricky beast. I expect 6model to bring a nice boost
for code that is OO-heavy and could benefit from native-typed attributes. I
am also convinced, and am prepared to prove, that some of the PCC refactors
I've been talking about lately will bring some performance improvements of
their own. JIT is, I think, the big mountain on the horizon, so anything we
do until then is probably peanuts in comparison. However, we can make real
progress now, if we aren't expecting to move the world.

In addition to these core changes, I think we are going to see a lot of work
happen throughout the ecosystem. We're seeing something of a bump with
relation to numerical computing projects, and we're also seeing several new
tools for compiler-building being exercised. We have Python and
JavaScript compilers in the works as part of GSoC, a perennial interest in a
Ruby compiler, growing interest in a PHP compiler, a new (and growing) R
compiler, and several other projects popping up in various places. Do not be
surprised in the least if, by the 4.0 release, one or more of these high level
languages is good enough for most general-purpose programming tasks for
Parrot. Also, and this goes without saying, expect to see Rakudo continue
progressing at a phenominal rate.

The next few months are going to be busy and hopefully exciting as well. I
think we're going to get in some very important new features and changes that
we've been wanting for a long time.
