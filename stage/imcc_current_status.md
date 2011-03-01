---
layout: post
categories: [Parrot, IMCC]
title: IMCC Refactor, Current Status
---

The biggest thing I've been working on for the past few release cycles in
Parrot world are the IMCC cleanups and refactors. I've done a hell of a lot of
work in several separate branches, the most recent of which is called
`whiteknight/imcc_compreg_pmc`. First I'm going to talk a little bit about
what I've been doing in this and previous branches, and then I'm doing to
talk about what the current status is. Finally, I'm going to talk about what
the plans are from here, and how this work fits in with some of our longer
term goals.

## Work So Far

The IMCC work proceeded in 6 general stages, thought they weren't always
cleanly sequential. These 6 stages were:

1. General IMCC cleanup. I moved all the previous interface functions into a
   single file (`compilers/imcc/main.c`) and started cleaning up the relevant
   code to be more readable. As I cleaned up, I started finding places where
   duplicated code could obviously be consolidated. I wrote about some of
   these areas [in a previous post][imcc_api_outline].
2. I removed the `imcc_info` field from the Parrot Interp structure. Instead
   of passing an interpreter reference as the first argument to every IMCC
   function, I changed it so that we were passing an `imc_info_t` structure
   instead. The `imcc_info_t` structure was modified to now contain a pointer
   to a parent interpreter (instead of the other way around).
3. I rewrote the public-facing interface functions, consolidating around a
   a single primary codepath. All compilations for files and strings now
   happens through a common sequence of functions, where only the top-most
   wrapper routines are different.
4. I created a new IMCCompiler PMC type to use as the new PIR compreg. At the
   moment it looks and acts a lot like the old NCI PMC we were using, but it
   contains state such as the current `imc_info_t` structure.
5. I created a new embedding API for IMCC, similar in design to Parrot's
   embedding API. These new functions are located in the new file
   (`compilers/imcc/api.c`), and are intended to be used in similar places and
   in similar ways to [Parrot's embedding API][parrot_embed_api].
6. I changed the way IMCC handles errors internally. Now it uses normal
   Parrot exceptions instead of using a custom exception system. I've only
   scratched the surface on this project, there is a lot more work to be done
   once we're ready.

[imcc_api_outline]: /2011/01/18/imcc_interface_functions.html
[parrot_embed_api]: /2010/11/26/embedding_api_home_stretch.html

A final result of all this work is that, besides the IMCCompiler PMC, there
are no direct ties from libparrot to IMCC. If IMCCompiler becomes a dynpmc
(and it will, eventually), libparrot and IMCC will be completely separate
entities.

Part of the problem we were having with IMCC is that it made a number of
assumptions about it being the only compiler in Parrot. It would use this
special arrangement to write global data directly to the Parrot Interp struct,
and Parrot, in turn, would expect to find these results there. This created an
inequitable situation where we couldn't easily have new compiler frontends
used in place of IMCC because IMCC was required at all stages. By separating
IMCC out and starting to erect proper encapsulation barriers between it and
libparrot, we both identify the necessary interface for compiler frontends
to be able to use (including new non-IMCC frontends) and we also can start
creating utilities which do not require IMCC.

## Where Are We Now?

Currently, the branch builds and runs on most platforms. On my machine it
passes all tests, though some of the more intensive NQP-based tests eat up a
lot of RAM. A lot as in "3 Gigabytes" of it. For parrot hackers on
memory-constrained computers, these tests fail outright with memory panic.

Why?

This took me a little while to track down, and it's a non-trivial though
straight-forward fix. First, take a quick look at [TT #1990][tt_1990] for
some background information.

IMCC outputs a `PackFile*` structure pointer. In my embedding API work, I
added wrapper routines to wrap those raw pointers into an UnManagedStruct
PMC. UnManagedStruct is basically a dumb wrapper object around a pointer, and
provides no real logic for introspecting or maintaining that pointer. One
thing in particular that UnManagedStruct does not provide is the ability to
mark its contents for GC.

Occasionally when you run Parrot, prior to the fix for TT #1990, GC would run
after the `PackFile*` was returned from IMCC, but *before* it was loaded into
the interpreter for execution. In that short window, the packfile was
essentially invisible to the GC and it gets collected. When Parrot attempts to
load and execute the packfile, BLAMO!.

So, as part of the TT #1990 fix, we add in several instructions to block the
GC from executing at certain parts of the program. The GC doesn't run when the
packfile is unprotected, and our data is safe from premature collection.
However, this creates a new problem.

IMCC is not GC safe. To my knowledge, it never has been. This means that to
use IMCC you must first disable GC, run the compilation, and then reenable
GC.

1. Block GC
2. Run IMCC
3. Unblock GC

Combine that with the TT #1990 fix, and you get this sequence:

1. Block GC
2. Block GC
3. Run IMCC
4. Unblock GC
5. Unblock GC

Block and unblock operations increment and decrement a counter respectively,
so they nest. If you block it two times, you need to unblock it two times
before it will run again. This seems pretty straight forward, but consider now
what happens if we throw an exception inside IMCC as a result of a syntax
error or something: IMCC doesn't know how many times the GC has been blocked
on the way in, so it only unblocks GC once on the way back out. The GC block
count is never decremented back to 0, the GC never runs again, and Parrot
slowly consumes all available memory before running into a memory panic.

Awesome.

Now I've explained the problem, what is the solution? Luckily, while I've been
tooling around in this IMCC branch Peter Lobsinger has been kicking butt and
taking names in some other areas of Parrot. Recently he added several new
pointer-based PMC types to replace UnManagedStruct and its ilk. One such PMC
type is `PtrObj`, which includes the ability to perform GC mark on its
contents.

As I mentioned above, the solution is straight-forward: Modify the IMCC code
so that we create a PtrObj PMC instead of an UnManagedStruct PMC to hold
the `PackFile*` pointer returned from IMCC. With that in place, we can remove
all the extra GC block instructions added in TT #1990 and hopefully start
talking about a merger shortly thereafter.

## Where To Go From Here

The IMCC work is a little bit tangential to some of our stated roadmap goals.
However, I do believe that doing this work is going to provide a necessary
boost, and help make some future work much easier. One of the areas where we
need to focus much of our attention is on the packfile system. That system
needs some radical improvements, the first step of which is properly
encapsulating it behind a useful API. Since IMCC is one of the primary
encapsulation violators of that system, cleaning IMCC up will help to push
along that work.
