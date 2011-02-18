The biggest thing I've been working on for this release cycle in Parrot world
are the IMCC cleanups and refactors. I've done a hell of a lot of work in
four separate branches, the most recent of which is called
`whiteknight/imcc_compreg_pmc`. First I'm going to talk a little bit about
what I've been doing in this and previous branches, and then I'm doing to
talk about what the current status is. Finally, I'm going to talk about what
the plans are from here, and how this work fits in with some of our longer
term goals.

## Work So Far

The IMCC work proceeded in 4 general stages, each of which had it's own
branch. These branches built off each other and occasionally were developed
in parallel.

1. General IMCC cleanup. I moved all the interface functions into a single
   file and started cleaning up the relevant code to be more readable. As I
   cleaned up, I started finding places where duplicated code could obviously
   be consolidated
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

Currently, the branch mostly builds and mostly runs. Parrot in the branch
builds and builds most of the libraries included in the standard make target,
but it segfaults for me in the later stages because of a data corruption
problem. I have not been able to track this problem down, although I have seen
that if I disable the garbage collector it does not occur. I don't know if the
GC itself is causing a problem, or if running the GC in a predictable way
changes the memory scheme and exposes an underlying issue. Memcheck doesn't
seem to return any interesting information, and fellow Parrot hacker
plobsing can't even reproduce the failures on his development machine. What
luck!

The branch is basically feature complete. Once we get the bugs sorted out,
fix things so all tests pass again, and make sure NQP and Rakudo build fine
against it, I think we will be good to merge. This certainly won't happen
before the 3.1 release, although I am hopeful that we can get it sorted out
for the 3.2 release in March.

If anybody reading this is decent with a debugger and wants to help, checkout
a copy of the code and give it a spin. I would very much appreciate any help
I can get with this project.

## Where To Go From Here

The IMCC work is a little bit tangential to some of our stated roadmap goals.
However, I do believe that doing this work is going to provide a necessary
boost, and help make some future work much easier. One of the areas where we
need to focus much of our attention is on the packfile system. That system
needs some radical improvements, the first step of which is properly
encapsulating it behind a useful API. Since IMCC is one of the primary
encapsulation violators of that system, cleaning IMCC up will help to push
along that work.
