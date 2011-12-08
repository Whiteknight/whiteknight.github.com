---
layout: post
categories: [Parrot, Advent2011]
title: Advent 2 - Winxed
---

For the second post in my almost-advent calendar, I'm going to talk about
Winxed.

Winxed, written by long-time Parrot hacker NotFound is quite an interesting
project and is definitely worth learning more about. It serves as something
as a diametrical opposite to NQP, the other lower-level Parrot language in
the ecosystem, but the two aren't competing. I think they are very
complimentary.

Many HLLs written on Parrot use, or probably should use, NQP and the
compiler-building features written into that language. Winxed, against the
grain, is written in itself using a home-brewed recursive descent parser. To
perform the bootstrapping, the stage 0 winxed compiler is written in C++.
Stage 0, a paired-down version of the language is used to compile the stage 1
compiler which is written in that subset. Stage 1 is used to compile stage 2,
which is a more full-featured version of the language. Stage 2 is used to
compile itself into stage 3. Stage 3 is what you and I use when we install
Winxed.

It sounds much more complicated than it is, but the net result is clear and
simple: Winxed written in Winxed. If you know Winxed, or are familiar with
some of the big languages it is inspired by (C++, Java, C#, JavaScript),
you'll be able to not only write software for Parrot, but also be able to
hack on the Winxed compiler itself. I've submitted a few patches and feature
additions, and I've found it to be a pleasure to work on.

Since bundling with Parrot in th 3.6.0 release, which incidentally is when
NotFound started keeping track of version numbers, the language has come a
long way: Various optimizations, a debugging mode with optional asserts
and conditionals, several new built-in functions, support for multiple
dispatch and most recently a new "inline" feature which allows you to inline
certain types of code for performance improvements by avoiding extra PCC
calls.

I don't know where NotFound is planning to take Winxed in 2012. I have a few
ideas for features I might like to propose and provide patches for such as
better syntaxes for parameters and arguments (instead of using PIR flags),
better code generation to account for some IMCC and PCC changes I want to make
in Parrot, and the ability to create packfiles directly using Parrot's
Packfile PMC types (without generating intermediate PIR and then using IMCC
to compile it). Winxed could help serve as a major driver of change if it
supports new features and semantics quickly, and closely tracks changes made
in Parrot that could potentially break code written directly in PIR.

I personally use Winxed to implement my Rosella project and the first stage of
my JavaScript compiler "Jaesop". Other people are using it as well. Expect
more code to be written in Winxed in 2012, more projects to use it, and many
bits of existing code to be rewritten in it.
