---
layout: post
categories: [Parrot, Embedding]
title: PackFile and IMCC Cleanup Branches
---

A few days ago I started a new branch to get rid of a function which
particularly irked me: `PackFile_new_dummy`. To get rid of it I started a new
branch appropriately called `kill_packfile_new_dummy`, and ripped that
function out completely. That was all I was planning to do before the 3.0
release tomorrow, but the more code I read, the more I felt like I needed to
do. A few more branches later I've made some nice cleanups to IMCC, the first
steps on what will turn out to be a very long road.

Parrot's packfile system is nothing if not a little bit ugly. Many of the
ideas in the subsystem are decent, however. The design of a platform-agnostic
bytecode format is good (but there are tickets open to make it even better).
The design which uses multiple segments in a modular file format is good. The
ability to abstract the file format details away behind a series of helpful
structures and API functions is good. The ability to swap bytecode objects in
and out of the interpreter without too many side-effects is good. In short,
there are lots of good ideas in this system, but what's lacking is the
implementation.

Even if the ideas are good there are too many problems with the
implementation: There is absolutely *zero* encapsulation of any of the related
structures, including Parrot's interpreter structure. The fields in the
interpreter structure which are related to packfiles are used pervasively
throughout the packfile system, but also throughout systems which need to make
use of packfiles (like IMCC). This is really not a great thing if we are
looking to have things like "stable APIs". In addition to the interpreter
struct, there are several Packfile-specific structures which are also not
encapsulated, which interact with each other in complex ways, and are not
particularly friendly to work with.

There isn't even so much as consistent naming of functions. The standard
naming convention for the Packfiles system is to prefix public functions with
`Parrot_pf_*`, but very few functions in the system use this convention. Some
functions are named `Parrot_pbc_*`. Some are named `PackFile_*`. Some are
named `PF_*`. Some are named according to no obvious convention at all. Weird
terminology is used throughout the system for various things. For example,
what is a "fixup"? What is a "pragma"? Damned if I know.

Most of these functions are located in the `src/packfile/` directory, though
some are located in `src/embed.c`, and a handful of other important
packfile-related routines are located in `src/interp/` and even in
`compilers/imcc/`.

In short it's a huge mess, and cleaning it up has become something of a
community focal point in the coming months. It's become a task that I
personally want to work on, even if it happens in small fits and spurts. 
Yesterday I talked about 
[some specific cleanup tasks I wanted to perform][packfile_cleanups], which
really only represented a small handful of the long-term improvements I think
we need here. The `kill_packfile_new_dummy` branch is one such spurt, one of
the first small babysteps on this path. It was followed shortly by the
`imcc_cleanups` branch (which I will be talking about here and in future
posts), the `packfile_write_api` branch, and at least two other branches
on my local machine that I haven't pushed out yet. Like I said before, just
the first baby steps in a very long road ahead.

[packfile_cleanup]: http://whiteknight.github.com/2011/01/15/packfile_changes_and_compilers.html

Parrot's interpreter structure has a field named `initial_pf` which, as far as
I can tell, holds the current PackFile structure and does not save the initial
one. The interp also has a field `code`, which I think is usually an alias for
`interp->initial_pf->seg`, but I don't know if that relationship always holds
true. I would hope so, or if not I hope the divergences are documented
somewhere (I doubt that too). IMCC, Parrot's burdensome PIR compiler,
frequently accesses both these fields directly. In fact, IMCC is designed to
write its compiled PackFiles directly to `interp->code`, making it nearly
impossible for Parrot to compile a program down into a new PackFile safely,
without creating problems in the currently-executing program. There's a huge
amount of code to try and juggle temporary values between these global fields,
and a not-insignificant number of bugs and TODO feature requests that arise
from it all.

Attempting to compile a string or a file with IMCC from PIR code requires far
too much handwaving, setup, and takedown logical nonsense. And even with all
that work there are still side-effects to be dealt with. To top that off, the
system is not nearly as flexible as anybody would like. You can take a string
of code, compile it, and get back a Sub PMC (or an Eval PMC, which is like the
really ugly older stepsister of Sub), but you can't get back a PackFile PMC or
write out a .pbc file. At the moment, There isn't even any way to ask IMCC
for these things.

You can load in a file of PIR code through IMCC, and it will run all the
`:load` functions in the file, but not the `:init` functions. It also won't
automatically execute the `:main` function, though it will return you a
reference to it. We would like the ability to specify what, if anything, gets
automatically executed, and we would like to be able to decide what gets
returned. This is all a little bit off topic, I'll have plenty more to write
about the evils of IMCC later.

`PackFile_new_dummy` is used to create a new empty PackFile and stash it in
`interp->initial_pf` and `interp->code`. This is useful for things (like IMCC)
which expect the interpreter to have a valid PackFile loaded, but may be
called before anything else. Later in the program, new PackFiles or parts of
PackFiles (segments) can be merged into `interp->initial_pf`. This creates,
essentially, a difference between a PackFile which is loaded first and a
PackFile which is loaded second (or thereafter). Take these two types of
packfiles (which are identical structures, at the binary level), and give them
different functions for working with them, and have different subsystems make
assumptions about which one is which without being told, and you end up with a
whole heap of trouble. This is what Parrot has right now.

Everybody treats Parrot's interpreter as their own personal information
dumping ground. And when you have multiple poorly-designed subsystems all
dumping crap and garbage to the same places, you end up with conflicts and
trouble.

My first step to correcting this was aimed at removing `PackFile_new_dummy`. I
removed that function, and fixed things up so that the differentiation between
initial and subsequent packfiles was almost completely internalized to the
packfile subsystem. I got some great help from Peter Lobsinger in all this.
Peter has become something of our resident PackFile expert, although I've been
reading and modifying code here like a madman, and may be slowly catching up
to him. This was a great start, but there's always more to do.

Since we can't merge anything before 3.0, and since I had a few more ideas, I
just kept working. I added a few more (properly named) accessor functions into
`src/packfile/api.c` to interact with `interp->initial_pf` and `interp->code`.
Then I jumped into IMCC and started using those accessors instead of the
direct struct references. This was the first stage of what became a pretty
major overhaul of `compilers/imcc/main.c`. This first stage is essentially
complete and then some, but it opens the way for a few new projects in the
near future.

I've got two new projects that I want to work on now.

1. I want to cleanup and unify the various interface functions for IMCC. I
   want it to have a proper call-in interface that abstracts out a few
   ugly details and eliminates a large amount of duplicate code. I'll be
   posting a blog post about this topic tomorrow or shortly thereafter.
2. I want to modify IMCC so that it no longer writes data directly to
   `interp->initial_pf` and `interp->code`. Instead, it is going to create a
   fresh packfile, write only to that, and return it. From there, Parrot can
   load it, merge it into it's current packfile, write it out to .pbc, or
   completely discard it without suffering any side-effects. This is the most
   important thing we can do, but will probably wait until after the interface
   work so that I have fewer code-paths to modify.
  
I've got a lot of big plans for the packfile system and IMCC, some of which
I'll be blogging about and committing fixes for in the coming days. When the
3.0 release comes out tomorrow I'm going to merge branches which are ready,
and continue working in new branches on related projects. 