---
layout: post
category: [Parrot]
title: The Plan for PIR
---

At PDS yesterday somebody asked me to clarify the long-term roadmap for PIR.
Some people, like fellow member of the Parrot Foundation board of directors
James Keenan, have suggested an information flow where people write opinion
pieces and unpolished work on blogs and elsewhere and eventually condense them
down to posts on the parrot-dev list and eventually to documentation in the
repository. This is the first leg of that race.

I've discussed, at length, the [problems with PIR as a language][pirproblems].
I can sum up the problems pretty well in a short list:

1. It's harder to parse for the machine than a normal assembly language. This
   leads to bad parsing performance and high parser complexity (see IMCC).
2. PIR doesn't really share any relationship with the underlying Parrot
   bytecode. A good assembly typically has a 1:1 relationship with the
   underlying machine code for fast and straightforward assembly/disassembly.
3. It's much harder for the average human programmer to use than an HLL, or
   even a medium-level systems language.
4. IMCC is the software equivalent of an unsanitary death trap. Maintaining
   it is a pain in the ass. Replacing it with something else is the paln, but
   won't be easy either.

I could go on but I'll leave it here. PIR is really the wrong solution for
the job and there's no real reason why it should be the one and only accepted
method to interact programmatically with Parrot. Parrot receives no benefit
from using PIR compared to some other language, and certainly receives no
benefit from treating PIR as the built-in *de facto* standard.

What exactly Parrot should be is open to much interpretation, and anybody who
has a vision of it probably has a different vision from everybody else. My
personal vision for it involves libparrot as a language-agnostic bytecode
interpreter and dynamic language runtime, a series of loadable compiler
modules for various languages, and a series of executable front-ends to
implement command-line access to compiler modules on libparrot.

In my vision libparrot becomes a *bytecode engine*, with the tools necessary
to execute bytecode, create bytecode, modify bytecode, and analyze bytecode.
It does all these things without making any assumptions about where that
bytecode came from, what language the human wrote originally to generate it,
or what kinds of syntax or semantics that language has. We would then have
a `parrot-pir` frontend executable which would link to libparrot, and load
in a PIR compiler to it (be it IMCC, [PIRC][] or [PIRATE][]). This is really
no different from our current fakecutables generated from pbc_to_exe: Those
create a small wrapper program which links with libparrot and registers a
customer compiler object to use as the front-end. `parrot-nqp` is a perfect
example of this.

I want people to be able to write compilers in their choice of language with 
their choice of technoloties. They can write parsers in flex/bison, or ANTLR, 
or PCT, or whatever. Then they write a small wrapper program, link to 
libparrot, register their compiler objects and everything just works. 
Compilers should be able to be written in C, C++, or any language which 
compiles down to native machine code, in addition to compilers which run on 
top of Parrot and are themselves availablein PBC. Assuming you can turn all of 
these things into a compiler PMC, it shouldn't matter to libparrot.

With this kind of mentality, libparrot becomes very small. It contains the
functionality necessary to interact with bytecode and some basics of a runtime
engine: method dispatch, exception handling, task scheduling, library/module
management, garbage collection, native call bindings, etc. Everything else
would be added in to libparrot as needed on a per-user basis.

Keep in mind that eventually Parrot is going to move to Lorito, a new bytecode
format (among other things). PIR moves from being a central part of Parrot's
execution strategy to just another overlay on top of Lorito. PIR ops,
currently written in C and compiled in to libparrot, are re-implemented in
Lorito.

PIR needs to come out of Parrot. It doesn't need to disappear entirely, but
it shouldn't be a built-in default standard anymore. Embedders should have
the option of including the PIR compiler or *excluding it*. Default behaviors
in libparrot which rely on PIR, or assume PIR need to be changed to be more
agnostic towards available input language.

We can replace the special status of PIR with the notion of a "default" 
compiler if people want, so that when we attempt to load in an arbitrary code 
file Parrot can attempt to compile it with the default compiler. For instance, 
the parrot-nqp binary could treat NQP as the default compiler, but could still 
include a secondary PIR compiler object for implementing the pir:: 
pseudonamespace and the Q:PIR inlining mechanism. When parrot-nqp attempts to 
load an arbitrary code file, it will assume that file is written in NQP and 
act accordingly.

PIR is one of those things where I feel like we can take it out and actually 
make Parrot *better*, more usable, and more useful. This is, of course, just 
an opinion piece, but I sincerely hope that some of these ideas make it into 
the mainstream of Parrot developer thought.
