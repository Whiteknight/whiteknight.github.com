---
layout: post
categories: [Parrot, IO]
title: More IO Work?
---

I might not be too bright. Either that or I might not have a great memory, or
maybe I'm just a glutton for punishment. Remember the big IO system rewrite I
completed only a few weeks ago? Remember how much of a huge hassle that turned
into and how burnt-out I got because of it? Apparently I don't because I'm back
at it again.

Parrot hacker brrt came to me with a problem: After the io_cleanup merge he
noticed that his mod_parrot project doesn't build and pass tests anymore. This
was sort of expected, he was relying on lots of specialized IO functionality and
I broke a lot of specialized IO functionality. Mea culpa. I had a few potential
fixes in mind, so I tossed around a few ideas with brrt, put together a few
small branches and think I've got the solution.

The problem, in a nutshell is this: In mod_parrot brrt was using a custom
Winxed object as an IO handle. By hijacking the standard input and output
handles he could convert requests on those handles into NCI calls to Apache and
all would just work as expected. However with the IO system rewrite, IO API
calls no longer redirect to method calls. Instead, they are dispatched to new
IO VTABLE function calls which handle the logic for individual types.

**First question**: How do we recreate brrt's custom functionality, by allowing
custom bytecode-level methods to implement core IO functionality for custom
user types?

**My Answer**: We add a new IO VTABLE, for "User" objects, which can redirect
low-level requests to PMC method calls.

**Second Question**: Okay, so how do we associate thisnew User IO VTABLE with
custom objects? Currently the `get_pointer_keyed_int` VTABLE is used to get
access to the handle's `IO_VTABLE*` structure, but bytecode-level objects cannot
use `get_pointer_keyed_int`.

**My Answer**: For most IO-related PMC types, the kind of `IO_VTABLE*` to use is
staticly associated with that type. Socket PMCs always use the Socket IO VTABLE.
StringHandle PMCs always use the StringHandle IO VTABLE, etc. So, we can use a
simple map to associate PMC types with specific IO VTABLEs. Any PMC type not in
this map can default to the User IO VTABLE, making everything "just work".

**Third Question**: Hold your horses, what do you mean "most" IO-related PMC
types have a static IO VTABLE? Which ones don't and how do we fix it?

**My Answer**: The big problem is the FileHandle PMC. Due to some legacy issues
the FileHandle PMC has two modes of operation: normal File IO and Pipe IO. I
guess these two ideas were conflated together long ago because internally the
details are kind of similar: Both files and pipes use file descriptors at the
OS level, and many of the library calls to use them are the same, so it makes
sense not to duplicate a lot of code. However, there are some nonsensical issues
that arise because Pipes and files are not the same: Files don't have a notion
of a "process ID" or an "exit status". Pipes don't have a notion of a "file
position" and cannot do methods like `seek` or `tell`. Parrot uses the `"p"`
mode specifier to tell a FileHandle to be in Pipe mode, which causes the IO
system to select a between either the File or the Pipe IO VTABLE for each call.
Instead of this terrible system, I suggest we separate out this logic into two
PMC types: FileHandle (which, as it's name suggests, operates on Files) and
Pipe. By breaking up this one type into two, we can statically map individual
IO VTABLEs to individual PMC types, and the system just works.

**Fourth Question**: Once we have these maps in place, how do we do IO with
user-defined objects?

**My Answer**: The User IO VTABLE will redirect low-level IO requests into
method calls on these PMCs. I'll break `IO_BUFFER*` pointers out into a new PMC
type of their own (IOBuffer) and users will be able to access and manipulate
these things from any level. We'll attach buffers to arbitrary PMCs using named
properties, which means we can attach buffers to *any PMC* that needs them.

So that's my chain of thought on how to solve this problem. I've put together
three branches to start working on this issue, but I don't want to get too
involved in this code until I get some buy-in from other developers. The
FileHandle/Pipe change is going to break some existing code, so I want to make
sure we're cool with this idea before we make breaking changes and need to patch
things like NQP and Rakudo. Here are the three branches I've started for this:

* `whiteknight/pipe_pmc`: This branch creates the new Pipe PMC type, separate
  from FileHandle. This is the breaking change that we need to make up front.
* `whiteknight/io_vtable_lookup`: This branch adds the new IOBuffer PMC type,
  implements the new IO VTABLE map, and implements the new properties-based
  logic for attaching buffers to PMCs.
* `whiteknight/io_userhandle`: This branch implements the new User IO VTABLE,
  which redirects IO requests to methods on PMC objects.

Like I said, these are all very rough drafts so far. All these three branches
build, but they don't necessarily pass all tests or look very pretty. If people
like what I'm doing and agree it's a good direction to go in, I'll continue work
in earnest and see where it takes us.
