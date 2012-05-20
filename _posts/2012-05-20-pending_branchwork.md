---
layout: post
categories: [Parrot]
title: Pending Branchwork
---

As I promised in my last post, I have several branches up in the air that need
to be worked on. Some branches merged last week after the release. Others
are pending to merge soon and some are still in development. In this post I'm
going to give a short summary of these things, since I haven't been posting
regular updates like normal.

### Already Merged

After the release last week I merged three small branches that brought small
changes and appeared to test cleanly with NQP and Rakudo. In short, these
were uncontroversial.

* `whiteknight/gh_675` named after the
  [Github Issue of the same name](https://github.com/parrot/parrot/issues/675),
  this branch removed the `can` vtable. In all cases in core and in external
  projects where I looked, the `can` vtable was simply a redirect to the
  `find_method` vtable and a check for null. There's no need for this added
  indirection, we can call the `find_method` VTABLE directly from `can`
  opcode.
* `whiteknight/imcc_file_line` This branch removed some very old,
  long-deprecated IMCC directives. The `.line` and `.file` directives were
  not poorly implemented (as far as IMCC goes) but they weren't used and
  weren't introspectable. The `setline` and `setfile` directives (yes, they
  are directives even though they looked like opcodes!) weren't used anywhere
  and weren't implemented well. I've removed all four. Now, we can use the
  `.annotate` directive to replace all of these and add other metadata besides
  in a way that is easy to introspect from within running bytecode.
* `whiteknight/remove_cmd_ops` removed a few command-line arguments from
  the parrot executable which were non-functional. These arguments have been
  disconnected since the time of the IMCC API cleanups months ago, and nobody
  had even noticed. Now they're gone.

Those things out of the way, here's a list of some of the branches that are
currently unmerged but may be merging soon.

### `eval_pmc`

This is one of the most disruptive branches I've got going, which is why I'm
in no hurry to merge it. Before I can merge it I need to patch both NQP and
Rakudo. I submitted patches for these but they weren't ready to apply and I
have to go back and re-do them.

This branch removes the deprecated `Eval` PMC. The `IMCCompiler` PMC has
already been updated to use a PDD31-compliant interface, which returns a
`PackfileView` PMC instead of an `Eval`. NQP and Rakudo need to be updated to
use this new interface instead of the older `VTABLE_invoke` one. This update
will work in the Parrot master branch just fine, so we can make those updates
to NQP and Rakudo and test them thoroughly before we merge the `eval_pmc`
branch in.

### `remove_sub_flags`

This is a much bigger and much more disruptive branch. However, because of the
fact that NQP and Rakudo don't really use subroutine flags for their
control flow, those two projects won't really be affected as much as everybody
else will be.

The `remove_sub_flags` branch removes the `:load` and `:init` flags from the
PIR syntax and replaces them with `:tag`. The only real way to work with
`:tag` is through the `PackfileView` PMC, so we need to merge the `eval_pmc`
branch into Parrot first before we can make any further progress on this one.
This is a back-burner task and will probably not be touched before the end of
the summer.

### `whiteknight/gc_finalize`

We've received some requests from Rakudo folks that we need to start getting
serious about GC finalization. This involves two changes: First is setting the
GC to perform a finalization sweep at interp exit, which it currently is not
doing. The second is to fix some sweep-related behaviors so the `destroy`
VTABLE can be much more sane and useful.

The `whiteknight/gc_finalize` branch does both of these things. First, it
re-enables GC finalization which had been turned off for so long that the
code for it no longer works in master. Second, it moves to a two-stage sweep
algorithm, so that we execute all `destroy` vtables first before we start
freeing any resources.

There are still going to be problems with `destroy` vtables however, and I'm
searching for solutions to these. Let me illustrate with a short example. We
call GC to sweep typically in response to a request for a new PMC when we have
none on the free list. If we have an item on the freelist, we return that
immediately and very quickly. If not, we invoke GC to try and free up some
headers (or allocate new ones from the OS).

Let's say we're programming in Rakudo Perl6 and we have an object with a
destructor. For the purposes of our example, it's a DB connection object. That
destructor needs to call a method on a Socket object connecting the client
program to the server. As everybody should be aware of now, calling a method
in Parrot itself is going to allocate a CallContext PMC.

However, we run into a small problem because we're in GC *because* we're out
of PMCs to allocate. So if we try to allocate a new PMC at this point I don't
know exactly what will happen but I can only imagine that the results would
not be good. At the worst case, we recursively call into GC which goes back to
sweeping, which re-executes finalizers, and we get into an infinite loop.

I won't go into all the details here, I've got another (long) post drafted
that discusses these and some other issues related to finalization. This
`whiteknight/gc_finalize` branch solves some of the first few problems but
there will be more to come after that.

### `whiteknight/gh_663`

The `singleton` designator for C-level PMCs has been deprecated for some time
now, and the `whiteknight/gh_663` branch intends to remove them.

Here's how singletons work in Parrot: The `get_pointer` and `set_pointer`
vtables are used to manage a single reference to an existing singleton PMC
if any. To get the PMC, we invoke the `get_pointer` vtable *without an
invocant PMC* (the only such occurance of a vtable invoked without an existing
PMC reference in the whole codebase that I am aware of). If it returns NULL,
a new header is created. If the new header is created, the `set_pointer`
vtable is called on the new object with itself as an argument.

This all happens inside `Parrot_pmc_new` and is mostly transparent, except
for the few bits of code throughout the system which violate this (rather
flimsy) encapsulation boundary.

The `get_pointer` and `set_pointer` vtables operate on `void*` pointers, so
we even lose typesafety. Plus, we don't expose `get_pointer` or `set_pointer`
vtables to PIR code, so there's absolutely no way to create a singleton
class at the user-level using this mechanism. You can do what users of all
other languages do and create an accessor and restricted constructor and
implement singletons that way. In fact, I think that's better.

The majority of offending code has been ripped out of this branch, though I'm
still seeing some segfaults during the build as a result of bad, unchecked
pointer accesses in places where encapsulation has been violoated. I've got
to spend a little bit more time tracking down some of these failures. Then,
assuming NQP and Rakudo aren't relying on this mechanism, the merge should be
relatively painless.

### `whiteknight/gh_610`

A while ago, moritz suggested that we improve integration of our ByteBuffer
PMC type, especially with our FileHandle and Socket types. We should be able
to read a sequence of raw bytes from either of those PMCs into a ByteBuffer
and we should be able to write raw bytes from a ByteBuffer into either of
those destinations too.

The `whiteknight/gh_610` aims to make this a reality. Already I've done most
of the code work to get this in place, though I haven't added all the
necessary tests and documentation. Plus, a few coding standards tests are
failing too.

While looking at this code, I am reminded that the IO subsystem is kind of
messy. I've tried to clean it up in the past, and made a few small
improvements over time. However, without a larger guiding vision to follow,
I never really had a great idea of what kind of larger architectural changes
to make to really bring this subsystem up out of the mud.  After working on
this branch, I finally had something like a flash of insight, and think I have
a good idea about how to clean things up. This leads me to...

### `whiteknight/io_cleanup1`

My idea is a relatively simple one: All our IO operations are controlled by
the various PMC types (FileHandle, Socket, StringHandle, etc), but all our
IO API functions are currently implemented as ugly (and brittle) switch
statements to pick between execution pathways for these different types. A
far better idea would be to separate out the different logic behind a
virtual function dispatch table (vtable).

I've written up some proposed changes in the `whiteknight/io_cleanup1` branch,
and will start work if other people think it's a decent idea.

The key points are as follows:

1. Move all FileHandle-specific logic into src/io/filehandle.c. Do the same
   for Pipe, Socket and StringHandle types.
2. Implement a new `io_vtable` type, which will contain a dispatch table for
   common operations. Each one of the files created in #1 above will implement
   the routines for one `io_vtable` and supporting logic.
3. Buffering will be refactored. Instead of the FileHandle PMC containing
   several attributes for buffering, we'll instead use an `io_buffer` object
   to hold buffering details. An encapsulated buffering API will take this
   buffer structure and the relevant vtable and automatically perform
   buffering if necessary.
4. I am going to start separating out Pipe logic from FileHandle, though I'm
   not planning to create a separate type for it quite yet.

Once these things are done, I think the IO system will be much cleaner and
much more hackable. This is lower priority right now until some of my ideas
are vetted, but I'm glad I finally have a plan in mind after so many years of
staring helplessly at this code.

### `whiteknight/sprintf_cleanup`

The engine for our `sprintf` implementation is sort of old and messy. It's
some very functional and very stable code, but it needs to be brought up to
date with our modern coding and organizational standards.

In the `whiteknight/sprintf_cleanup` branch I make several changes, most of
which are entirely internal and should not affect users at all:

1. I move the files from 'src/misc.c` and `src/spf_*.c` to
   `src/string/sprintf.c` and `src/string/spf_*.c` respectively.
2. I've cleaned up some header-file nonsense and created a new
   `src/string/spf_private.h` header file to hold private data.
3. I've changed the code to use a StringBuilder instead of the older (and
   now-incorrect) repeated string concatenations. With immutable strings,
   each concat operation creates a new STRING instead of appending to the
   pre-allocated buffer, which is extremely wasteful. I haven't benchmarked
   this change, but I suspect it has higher performance on longer, more
   complicated formats.
4. I've fixed a sub-optimal error message at request of benabik in
   [ticket #759](https://github.com/parrot/parrot/issues/759).

This branch is almost complete and I'll probably merge it this weekend.
Besides the text of the exception message, there are no visible user changes
so it shouldn't be controversial at all.
