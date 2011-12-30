---
layout: post
categories: [Parrot, Advent2011]
title: Advent 10 - PACT
---

Since I've been a member of the project Parrot has had PCT, the compiler
toolkit. PCT is a pretty great library that uses an abstract syntax tree (PAST)
and ultimately converts that into PIR code and then Parrot bytecode.

There are two problems with PCT which are becoming more painful as time goes on.
The first is that PCT uses PIR as an intermediary instead of attempting to
generate PBC directly. The second is that PIR is structured in a fundamentally
different way from PBC, so there isn't even a straight-forward way to update
PCT to generate PBC directly. At least, not a good way to do it. Because the
assumption is that PCT generates PIR code, there are assumptions made towards
that end at all layers of abstraction and even literal PIR code strings included
in various places that are going to be hard to rid ourselves of.

Besides those two fundamental problems we would also like to see a compiler
toolkit that is more full-featured. Specifically we would like to include an
engine for tree transformations and optimizations included directly in the
toolkit instead of being an optional add-on.

Benabik has been working on an idea for a rewrite of PCT called PACT which will
be designed from the beginning to generate bytecode instead of PIR. The new
library, if the ideas pan out, should be faster than PCT and should generate
PBC directly instead of generating PIR. There's only one problem: We haven't
written it yet.

Parrot has a series of PMC types for generating packfiles at a very low level.
The first task for PACT is to create a more usable interface for these PMC types
so we can create packfiles in a more straightforward way. We've drafted some
code towards this end already, but everything is in the prototype stage so far.
Starting in January we are going to finish building this base layer and then
we'll be building upwards from that.

A low-level syntax tree ("LLST", for now) layer will flesh out many of the
primatives and tree structures, and will interface with the foundation layer to
build working packfiles. Benabik is talking about a new "PACT Assembly" language
which we can use for low-level testing and proof-of-concept tools. Essentially
we want something low-level which can be translated into this LLST format
easily.

Above that will be a more generic abstract syntax tree type. This AST will need
to represent high-level semantics, and be amenable to optimization and
transformation to LLST. Once we have that, and I'm hoping we can have it in the
first half of 2012, we should be able to start using PACT as the backend for
various compilers. I'm specifically interested in getting Jaesop working with
PACT as quickly as possible.

PACT is also going to want to host a few other tools. Reflection and packfile
disassembly come to mind as things that we want. Some of these things I've been
prototyping under the auspices of Rosella, but eventually PACT should become the
one-stop shop for anything that a compiler might need to help generate or work
with bytecode. Tools that analyze bytecode might also be able to benefit from
the work.
