LAPACK (for PLA)
GSL
GMP
Other? (Cairo seems like a good choice)


## Problem Statement

This is a meta-task for several individual tasks which are written about on
the wiki and several hundreds of other possible projects which we haven't
written about.

The basic idea is simple: Pick a cool existing library and write Parrot
bindings for it. Some of the ideas which have been explicitly mentioned are
GSL, GMP and LAPACK (for Parrot-Linear-Algebra). If you can think of another
good, popular, useful library to do instead, do that.

## Difficulty

The difficulty for this project can range from extremely easy up till
extremely hard, depending on the library you pick and the method you use to
bind to it. Basic NCI bindings over a small library is very easy. More
involved bindings with things like dynpmcs or dynops, or libraries which
require lots of library code to marshall access to the interface routines
is a lot harder.

We want not just to provide access to native libraries, but to provide a
useful programming interface to them.

## Deliverables

To really be successful, you should be able to deliver the working bindings
and any necessary supporting code: Build infrastructure if anything needs to
be built, unit testing, example code, and documentation. Depending on what
the bindings look like and how easy they are to use, you may need to provide
significantly more examples and documentation.

## How to Get Started

First step is to pick a library for which Parrot doesn't have existing
bindings (or, doesn't have them in working condition). Next step is to start
getting into contact with the Parrot community over the mailing list and on
IRC. Talk to people who may know something about your target library and
Parrot's system for binding to libraries: myself, dukeleto, and plobsing are
all good first choices.

## Who Should Apply

This project is one which obviously has the most options. Don't consider this
if you get overwhelmed with too many possibilities. You need to pick an
existing native library which you are familiar with and for which meaningful
bindings can be written over the course of a summer. Picking the right library
and the right method for binding is non-trivial, and that needs to be done
before the summer even starts!

Chances are good that you're going to be writing, or at least reading, a lot
of C code for this project. You should be at least decent with C.
