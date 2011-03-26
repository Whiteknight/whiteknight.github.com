---
layout: post
categories: [Parrot, GSOC, Python]
title: GSoC Idea - Python 3 Compiler
---

## Problem Statement

This one is very straight-forward:

> Implementation of Python3 on Parrot, written in pure python as much as
> possible, that targets PIR or Winxed.

Simple: Parrot wants a compiler for Python 3.

### Alternate Ideas

Here is a short list of alternate project ideas related to Python. I'm not
going to go into depth about all of them, but if you love Python and want
to do something Python related, you might be able to find the glimmer of an
idea here:

* Instead of Python 3, build a compiler for Python 2.
* Create a wrapper API layer for loading and using existing Python extensions
  from Parrot. (This is probably a doozie of a project)

## Difficulty

*Difficulty*: 2/5 - 5/5

As I mentioned in the post about the JavaScript compiler project, creating a
compiler is not a trivial task no matter what language you use. Like the
JavaScript project, there are opportunities to reuse code: You could create
a PIR code generator for an existing Python compiler like PyPy.

There is a big difficulty range here, and that owes to there being several
options you can pursue. For instance, you could simply write a code generator
for an existing Python compiler and bootstrap. That's great, but not probably
not very hard and probably not enough to fill an entire summer, especially if
you're a great Python coder.

There is also going to be a huge impedance mismatch between the Python object
model and Parrot's object model. A "good" Python-on-Parrot compiler is going
to want to smooth out those details, and probably write a custom object
metamodel to use. A "working" Python compiler is not nearly as wonderful as
a "good" or a "fast" python compiler would be.

## Deliverables

At a basic level, we need a working python compiler which generates PIR (or
NQP, Winxed, etc). The compiler should generate correct, working code for all
of Python 3, or at least a very large subset of it.

Of course, most existing python compilers are for Python 2, not Python 3.
Parrot would love to have compilers for both languages, but if we have to
pick only one to focus on right now it would be Python 3. We do like the idea
of language bootstrapping, but there's no particular reason why you couldn't
have a Python 3 compiler written in Python 2. Really, you can write it in
whatever you want, the basic idea is that if you write your python compiler
in something that python coders like to code in (which, I suspect for no
apparent reason, is python), you'll gather attention from more python coders
to help out with it.

In addition to that, all the standard accoutrements are required: a build
system, unit tests, code examples, and documentation.

## Alternatives

If you don't want to create a complete bootstrapped Python compiler, an
alternative project idea is to create a compiler for a python subset language,
similar to how NQP is a subset of Perl6. The idea is that we can implement
a subset of python directly in PIR/Winxed/NQP, and use that to bootstrap a
python compiler without needing to rely on an existing python compiler.

## How to Get Started

If you are interested in this project (and at least one student has expressed
significant interest in it already) you should get onto the parrot mailing
list or IRC. You should probably take a look at the existing Pynie project and
get in touch with some of its developers like Allison.

## Who Should Apply

If you are interested in taking on a big open-ended language project, know
Python, and have some background with compilers this is a project for you. If
those things don't describe you, don't even attempt this one.
