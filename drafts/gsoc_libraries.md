---
layout: post
categories: [Parrot, GSOC]
title: GSOC Idea - Library Bindings
---

## Problem Statement

This is a meta-task for several individual tasks which are written about on
the wiki and several hundreds of other possible projects which we haven't
written about.

The basic idea is simple: Pick a cool existing library and write Parrot
bindings for it. Some of the ideas which have been explicitly mentioned are
GSL, GMP and LAPACK (for Parrot-Linear-Algebra). If you can think of another
good, popular, useful library to do instead, do that.

## Difficulty

*Difficulty : 1/5 - 4/5

The difficulty for this project can range from extremely easy up till
extremely hard, depending on the library you pick and the method you use to
bind to it. Basic NCI bindings over a small library is very easy. More
involved bindings with things like dynpmcs or dynops, or libraries which
require lots of library code to marshall access to the interface routines
is a lot harder.

We want not just to provide access to native libraries, but to provide a
useful programming interface to them. This may be wrapper classes, or other
utilities to make working with the native library not only possible, but
actually easy to do.

Library bindings should be language agnostic. That is, you should write the
bindings in such a way that they can be used from Rakudo, Cardinal, Partcl,
NQP, Winxed, and any other languages which run on Parrot.

For good measure, you could even write modules for specific HLLs too. As one
example, if your project is to create GMP bindings, you could create a basic
wrapper project which is language agnostic, and then use that project to
implement a Math::GMP module for Rakudo, as a proof of concept.

This is just one idea, there are many possibilities here, depending on which
libraries you choose and how you want to prensent them.

### Possible Libraries

There are three libraries mentioned by name in the tasklist on the Parrot
wiki:

* GSL
* GMP
* LAPACK (As part of Parrot-Linear-Algebra)

Other libraries and library families which might be useful, and some binding
ideas for them, are:

* gobject : implement a plug-in object model which uses gobject as its base.
  Or, simply provide native wrappers for gobject, without replacing the whole
  metamodel.
* Cairo, libgimp, OpenCl, etc : Graphics support on Parrot would be totally
  boss. We had OpenCl bindings at one point but they are in a state of
  disrepair.
* Tk, Gtk2, Qt, etc : GUI toolkits are going to be some of the hardest to
  write high-quality bindings for, but if you could get off to a good start
  with it and create a project for other people to get involved with, it's
  doable.

And while we're at it, there's no real reason why you would have to be
restricted to just existing native libraries. Adding low-level interfaces to
a variety of existing services and software would all be a good thing:

* Google APIs
* DBUS
* Amazon EC2 APIs
* Github APIs
* A Database server (MySQL, etc. You would need to write up the query
  generators, response parsers, and custom data types to hold resultsets, but
  it would be well worth it).
* CUDA, or other GPU-based accelerators

Again, just a handful of ideas. All of these would be a benefit to the
Parrot community, especially if done correctly.

## Deliverables

To really be successful, you should be able to deliver the working bindings
and any necessary supporting code: Build infrastructure if anything needs to
be built, unit testing, example code, and documentation. Depending on what
the bindings look like and how easy they are to use, you may need to provide
significantly more examples and documentation. Again, much of this project
is open to interpretation and the exact things that you will need to deliver
depends entirely on what you propose to do. No matter what you propose, you
will need to have some of the basics (unit tests, examples, docs, etc), but
the rest is up to you.

## How to Get Started

First step is to pick a library for which Parrot doesn't have existing
bindings (or, doesn't have them in working condition). Next step is to start
getting into contact with the Parrot community over the mailing list and on
IRC. Talk to people who may know something about your target library and
Parrot's system for binding to libraries: myself, dukeleto, and plobsing are
all good first choices. At the very least, one of us can point you in the
right direction.

## Who Should Apply

This project is one which obviously has the most options. Don't consider this
if you get overwhelmed with too many possibilities. You need to pick an
existing native library or even a family of related libraries which you are
familiar with and for which meaningful bindings can be written over the course
of a summer. Picking the right library and the right method for binding is
non-trivial, and that needs to be done before you even submit a proposal!

Chances are good that you're going to be writing, or at least reading, a lot
of C code for this project. You should be at least decent with C. You're also
going to need to write some code in a low-level Parrot language (PIR, NQP,
Winxed), and maybe wrapper code in an HLL of choice (Rakudo Perl 6, Cardinal,
Partcl, etc).
