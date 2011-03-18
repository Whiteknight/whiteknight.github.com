
## Problem Statement

This one is very straight-forward:

> Implementation of Python3 on Parrot, written in pure python as much as
> possible, that targets PIR or Winxed.

Simple: Parrot wants a compiler for Python 3.

## Difficulty

As I mentioned in the post about the JavaScript compiler project, creating a
compiler is not a trivial task no matter what language you use. Like the
JavaScript project, there are opportunities to reuse code: You could create
a PIR code generator for an existing Python compiler like PyPy.

In addition to the basic compiler, you are going to need to work on the
object model as well. The Python object model does not map well to the
default Parrot object model, and you're going to have to do some massaging
(at least) to get the object semantics to work. If you want the object system
to work *well*, you're going to have to do even more work.

## Deliverables

At a basic level, we need a working python compiler which generates PIR (or
NQP, Winxed, etc). The compiler should generate correct, working code for all
of Python, or at least a very large subset of it.

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
