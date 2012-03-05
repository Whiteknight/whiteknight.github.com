---
layout: post
categories: [Parrot, Packfiles]
title: Packfile Tags Cleanup
---

I've complained a lot about Parrot's packfiles and packfile-related subsystems.
A few months ago I started
[writing about a plan to fix some of these issues][load_init_1]. Now, after
something of an extended hiatus from hardcore parrot hacking, I've decided to
get started on doing all this work.

[load_init_1]: /2011/07/07/packfiles_work_continues.html

The biggest part of my plan, from a user-interface perspective was changing
this:

    .sub 'foo' :load
        ...
    .end

to this:

    .sub 'foo' :tag('load')
        ...
    .end

Also, as part of this change, we want to change this:

    load_bytecode "bar.pbc"

to this:

    $P0 = load_bytecode "bar.pbc"
    $P1 = $P0.'subs_by_tag'("load")
    $I0 = $P1.'is_initialized'('load')
    if $I0 goto done_initialization
    $P2 = iter $P1
    loop_top:
    unless $P2 goto loop_bottom
    $P3 = shift $P2
    $P3()
    goto loop_top
    loop_bottom:
    $P1.mark_initialized("load")
    done_initialization:

I've discussed these changes before but I will repeat one important detail that
is well worth repeating: Despite the fact that the total amount of PIR code
increases, the overall performance profile *improves*. The second set of code
snippets are faster to execute and are much more flexible. The performance
improves as the number of `:load` or `:init` subs in your packfile increases.
You gain the ability to tag subroutines with any string name you want, use as
many different string names for subroutine groups as you want, get a list of
subs to execute in any order at any time, be able to initialize and reinitialize
libraries at any time as many times as you want, and do a few other cool and
empowering things. Actually, the system is set up to allow any arbitrary PMC
in the constants table to be tagged with a string name and accessed using that
name, but the only thing we have syntax for right now are the Subs.

Another point that I also feel like repeating is this: People shouldn't write
PIR code by hand. Don't do it. You will write something friendly like Rakudo
Perl 6, or NQP or Winxed, and those compilers will automatically generate the
necessary PIR sequences for you. It will take some effort on my part to get
the various compilers updated, but the end user shouldn't see any differences.
So long as you aren't writing your code in PIR directly, you'll be fine.

So what's involved in making this change? The packfile system, especially the
packfile loader and the bits that are intertwined with IMCC are some of the
messier or at least most obscure parts of Parrot. This is one of the subsystems
of Parrot that is the least abstracted and requires the most low-level bit
twidling. The GC is a comparable beast in this regard. Things like memory
alignment, offsets, complicated nested `struct` definitions and accesses, byte
ordering, and a variety of other similar concepts are very common in this
subsystem. Any changes must be made with extreme care.

At the time of this writing I've already made several major changes in my branch
and have several more to go. I've removed the `:load` and `:init` flag syntax
from the IMCC lexer and parser (they've been deprecated for a long time now and
a replacement syntax has been available for some months). I've removed the awful
`do_sub_pragmas` and `PackFile_fixup_subs` functions and rearranged significant
amounts of logic. I've also taken a shotgun approach to converting `:load` and
`:init` flags in the various PIR code libraries to `:tag("load")` and
`:tag("init")` respectively. Some of these conversions were automated and a
little bit over zealous, but that is to be expected at this stage of the game.
Code for handling `:immediate` and `:postcomp` flags have been moved into IMCC
where they belong. libparrot now has no direct knowledge of either of those two
things.

Coming up are several more big changes before this branch is even remotely
close to being mergable:

1. I need to clean up interp initialization logic that had been spread around
   in various places. The embedding API defines a central execution path for
   running bytecode programs, and the initialization logic needs to be moved
   higher up along that path.
2. I need to set a reference to the owning PackfileView PMC in each loaded Sub
   PMC. Currently we anchor every single PackfileView PMC to prevent them from
   ever being collected by GC. By storing a reference to the Packfile in the Sub
   PMCs, we can mark reachable Subs, mark reachable Packfiles, and therefore
   allow old, unreferenced packfiles to be GC collected. This fixes an old and
   sizable memory leak (especially in programs that do a lot of dynamic
   compilation. If we do this in the right way in the right places, we can close
   the memory leak and reduce iteration over packfile constants looking for
   Subs and PMCs to mark and keep track of.
3. I need to rewrite `Parrot_load_language` and friends. This is the function
   that implements the `load_language` opcode behavior. A "language" in this
   context is a library package which usually contains a compiler object, a
   runtime, and any dynpmc and dynop libraries required by those. This codepath
   shared much logic with `Parrot_load_bytecode` (the guts behind the old
   `load_bytecode_s` opcode variant), and therefore shares many problems and
   misbehaviors.
4. I want to refactor the relationship between the `PackFile*` structure and the
   `PackfileView` PMC. If the PMC is the GCable wrapper around the structure,
   the whole thing is more easily managed by GC if we can guarantee that the
   PMC always exists if the struct does. Refactors here can further clean up
   code in the packfile loader and elsewhere.
5. We need to rip out a lot of old, dead, crufty code that is no longer needed.
   This includes things like `Parrot_compile_file` and `Parrot_compile_string`,
   which are unnecessary internally because we have easily accessible and easily
   usable compiler objects at the PIR level. This also includes various helper
   routines for `Parrot_load_bytecode`, `Parrot_load_language` and
   `do_sub_pragmas`, helper routines for working with `:load` and `:init` flags
6. I need to update Winxed, NQP-rx, NQP, Rakudo and any other HLLs and libraries
   that still rely on the old behaviors.

So there are plenty of steps left before we can talk merger, and I don't expect
to get this work anywhere near master before 4.2. It's a lot of work but it's
fun and will be rewarding when it's all done. I'm looking forward to getting
this all wrapped up in the coming weeks and months.
