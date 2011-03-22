## Problem Statement

This idea was written across two different tasks on the wiki:

> Create an LALR parser generator tool. The tool should be written in one of
> the lower-level languages which run on Parrot (PIR, NQP, Winxed, other) for
> maximum portability. The parser generator should take a language
> specification and compile it into an LALR parser in a manner similar to
> YACC/Bison. The generator parser likewise can be in any suitable language.
> For this project, a working LALR parser generator engine is more important
> than the ability to understand a huge number of input patterns and pattern
> modifiers.

> Create a new code generation backend for ANTLR to produce parsers which run
> on Parrot. Target languages can be any language which runs on Parrot,
> although lower-level languages are preferred for performance and portability
> (PIR, NQP, Winxed, etc). The backend should be able to create working code
> for the full range of ANTLR features.

Basically, the idea is this: LALR parser generators (such as YACC or Bison)
and the ANTLR parser generator are common and familiar tools for many
programmers. Many existing programming languages have parsers written using
these tools. We would like to be able to have these technologies running on
Parrot.

## Difficulty

*Difficulty : 2/5 - 5/5

Creating a backend for ANTLR is probably less difficult. Creating a full
LALR parser generator from scratch is probably a lot more difficult. There is
a lot of documentation available, and many working examples you could model
on, but it's still no easy task.

Notice that the Parrot ecosystem already does provide some parser tools:
NQP-rx (and the new NQP) have built-in recursive descent parsers and regexes.
Likewise, the project [ohm-eta-wink-xed][] provides an OMeta-based parser
for Winxed. The big benefit to having an LALR parser generator or an ANTLR
backend is to provide something which is more familiar to programmers, and
something which may help to facilitate migrations of various programming
languages to Parrot.

In both cases, the code you generate should be a low-level Parrot language
such as NQP, PIR, or Winxed.

## Deliverables

What we want to see is a working parser tool. In the case of an LALR parser
generator, you're probably going to provide more of a framework than the
actual working parser generator. In this case you have to write something
which is easy to hack on, because it's likely other people are going to be
continuing the work after the summer is over (you can help to, but once the
flood gates open you probably won't be alone anymore).

In the case of the ANTLR parser generator, we probably want to see a
completely functional backend, which can generate correct code for a variety
of ANTLR inputs.

In both cases we will want all the standard software accoutrements: ample
documentation, code examples, a build infrastructure, and a unit test suite.

## How to Get Started

You're also going to want to be familiar with the particular parser generator
tool you're working with. Check out Bison or ANTLR, read the docs, check out
existing examples, and try to figure out what's going on.

There are a couple parts to this project: Figuring out exactly what you want
to do (ANTLR or LALR), figure out what target language you will be generating
code for, etc.

## Who Should Apply

Parsers are one of the areas of computer science which tend to be the most
abstract and theoretical. You typically will find yourself following
algorithms which you do not completely understand. This is not to say that
you won't be able to understand things if you study up, but it's a realm
of coding that not everybody is comfortable in.

You should be familiar, at least to some degree, with parsers and the theory
of parsing. Since you're going to be writing one program which writes other
programs, you're going to want to be familiar with the idea of
metaprogramming and other mind-bending subjects.

Depending on the details of this project, you should be decent with C or Java,
at least decent enough to read through some complicated existing source code.
