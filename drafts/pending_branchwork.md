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
in no hurry to merge it. Before I can merge it, I need to patch both NQP and
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

### `whiteknight/gc_two_stage_sweep`
### `whiteknight/gh_610`
### `whiteknight/gh_663`
### `whiteknight/io_cleanup1`
### `whiteknight/sprintf_cleanup`
