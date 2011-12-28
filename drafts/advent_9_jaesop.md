---
layout: post
categories: [Parrot, Advent2011]
title: Advent 9 - Jaesop
---

Over the summer of 2011 we had a GSOC project that was trying to build a
JavaScript compiler for Parrot. That project made some interesting progress but
ultimately I think the approach ended up being flawed. That project attempted to
convert JavaScript into PIR code, which is a pretty big gap in terms of both
syntax and semantics.

Towards the end of the summer, after it was too late to go back and start from
the beginning, I had a different idea: What if we tried to generate Winxed code
instead of PIR code? Winxed would handle things like register allocation, and
the Winxed syntax is so similar in some places that the translation could
almost be verbatim copying. I put those ideas together with node.js and the
cafe compiler and Jaesop was born.

Jaesop intends to be a full JavaScript on Parrot compiler. The plan is to write
as much of it in JavaScript itself as possible, and bootstrap upwards. The
first component, "stage 0" uses node.js to run a converter that compiles
Javascript code into winxed. That's the only part we have working right now, but
what we do have is working pretty well.

In 2012 I want to get the next component, "stage 1" working. Stage 1 will use
the stage 0 compiler to compile itself. It will be written in JavaScript
(translated to Winxed en passant) and will be running entirely on Parrot. My
goal for stage 1 is to be generating bytecode directly instead of generating
either Winxed or PIR as an intermediary. That's going to require some serious
help from a compiler toolkit of some variety and I have very high hopes for
using PACT for this purpose when it's ready.

Stage 0 works very well and passes a small but interesting test suite I've set
up for it. It does not self-compile, but there are only a few relatively small
things standing in the way. For instance, regular expression support and pcre
bindings are not complete yet, and grammar currently requires semicolons at the
end of statements but the code generated from the grammar by Jison does not
always contain semicolons. I also haven't built in the `require()` function,
which is used by the stage 0 compiler to load in the various code files. These
are all small issues and with a small amount of work I expect stage 0 to be able
to self-host. Whether I want to expend that effort or focus attention on stage 1
instead is a different question entirely.

In 2012, once PACT has matured and 6model has been integrated into Parrot I
expect to get back to work aggressively on Jaesop. I'm looking for helpers too,
in case anybody reading this wants to get involved in the development of a new
JavaScript compiler.
