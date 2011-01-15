---
layout: post
categories: [Parrot, Packfiles]
title: Packfile Changes and new Compilers
---

PackFiles are going to be getting a pretty big upgrade in the coming months.
In fact, some of the big changes are already happening. As much as I can I
want to help with this effort and talk about it so people know where we are
going and can start getting excited too.

PackFiles in Parrot right now are an assortment of various low-level C
structures. These structures represent an in-memory representation of the
contents of a .pbc file. These structures interact in complex ways and are not
friendly to work with. Admittedly they could become much more friendly to work
with if their logic was properly encapsulated behind a nice API, but it isn't
and they aren't.

The immediate plan is to replace public-facing use of the packfile structures
with a new set of PackFile PMCs. From PIR code or through the embedding API
you'll be able to get access to these PMCs, and then operate on them how you
would any other PMC: through various VTABLE operations or through method
calls. I prefer method calls since VTABLEs represent a very rigid interface
which wasn't designed with PackFiles in mind, but they are much faster to
execute than methods. The packfile structures will probably still exist, but
will be used internaly only. They will be used in places where access
performance is far more important than a friendly, stable, and robust
interface.

This is going to require a few changes, however. One of the first things that
needs to change are our HLL compilers.

We don't have anything like a common interface for HLL compilers. All
compilers are made accessible as PMCs through the `compreg` opcode. During
initialization a program can register a compiler by name with the `compreg`
op, and later can retrieve an instance to it through the same op.

    compreg "foo", $P0
    $P1 = compreg "foo"

Internally they are actually two separate ops with two different signatures,
but to Joe Schmoe PIR user they look the same. If given my druthers they might
be named differently, but it's not worth the effort at this point to gain a
little bit more clarity here. Actually, if I had my way they wouldn't be ops
at all, but instead a method on the interpreter PMC.

Once we have a compiler PMC, what do we do with it? The answer isn't as simple
as "call the compile method and get the result". I wish it was. The answer
really depends on which compiler object it is. Some compiler objects, like
those based on HLLCompiler, are objects with a standard set of methods:
`.'compile'()` and things of that nature. Some, like the PIR compiler for
IMCC, are basic NCI function wrappers and are invoked directly:

    $P0 = compreg "PIR"
    $P1 = $P0(foo)

Here, `$P1` is a weird `Eval` PMC, something like a cross between a Packfile
PMC and an array of Subs. There are a few changes I would like to make here.

First, there's no reason why the PIR compreg should be a simple NCI wrapper.
It should be a full-fledged object with methods to do more than just compile a
string into an Eval PMC. I should be able to compile either a string of code
or a file of code and return a PackFile from them. I should also be able to do
an immediate compile+execute of a small code snippet, and an immediate execute
of a code file. I created a ticket a few days ago to deprecate the current PIR
compreg, and have it replaced with a new PMC type. I've started work on such a
new PMC, but haven't pushed it public yet.

Next, as I alluded to above, we don't want to be using Eval PMCs here. They
are hackish and ugly, and don't provide any real benefit. We should return a
PackFile PMC here, and be able to use the PackFile PMC for all our operations,
including searching for individual subs or executing it directly as a nested
stand-alone program. I've also opened a new ticket to have the Eval PMC
deprecated entirely and to replace it with a PackFile PMC in all cases. I
haven't started any serious work on this project yet, but I know other people
have been doing work in this direction.

