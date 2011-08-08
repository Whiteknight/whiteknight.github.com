---
layout: post
categories: [Parrot]
title: Parrot in Parrot, An Update
---

On a personal note, the house we were buying has fallen through. I'll spare
the details (the full telling of which involves substantial use of curse words
and unfounded allegations of illegal sexual deviancy). Time I was planning to
use hacking on Parrot from the comfort of my own house is now going to be
spent frantically searching the neighborhood for a new house to buy.

In a [previous post][tag_post] I talked about some of the work I was doing
with a new `:tag` syntax. The `:tag` syntax, and its underlying mechanisms
is intended to replace `:load` and `:init` flags (and maybe, eventually,
others like `:main` and `:postcomp` also) with something much more usable
and flexible. A big benefit here is that I can use the new PackfileView PMC
type to look up Sub PMCs by tag from running PIR code, and execute them
directly from PIR, without needing the current mechanism of nested runloops to
do it. As I have [shown in previous posts][nested_runloops_post], using nested
runloops brings a non-trivial performance penalty that we should try to avoid.

[tag_post]: /2011/07/23/imcc_tag_syntax.html
[nested_runloops_post]: /2011/05/10/timings_vtable_overrides.html

In Parrot semantics, the `:init` subroutines are supposed to execute *before*
the `:main` Sub executes. This is important because the `:init` subroutines
are typically used to set up things used by the main program, such as class
definitions and global data stores. However, the only way to currently
guarantee that the `:init` subs all execute before the `:main` sub is to
execute them separately, each in their own runloop. Basically, we have C code
to execute all the `:init` Subs in a loop, and then we find and execute the
`:main` Sub. If we want to consolidate and do everything from inside a single
runloop, one super-main routine needs to call both the `:init` and `:main`
subs together. This means we need to jump into some kind of standard PIR
entryway routine earlier in the startup process. I talked about this kind of
system [in a post several months ago][new_frontend_post], and how such a
system would look.

[new_frontend_post]: /2011/01/20/parrot_in_parrot_new_frontend.html

My hypothesis is this: If we had a different frontend program that jumped into
PIR code as early as possible and did as much processing there as could be
done, there are startup performance gains to be had. Embedding API calls,
because they need to set up things like error-handling mechanisms and other
call-in details, have overhead. Trying to do lots of work through the
embedding API was never the intended use of it. Instead, the embedding API
tries to expose the tools necessary to jump in to PBC execution, which is
where the real power and performance of Parrot is made available. Minimizing
the number of embedding API calls that we need to execute prior to user
program execution is a good thing. Identical operations can be done from PIR
without the call-in and call-out overhead of the embedding API. Also,
minimizing the execution of Parrot code in separate runloops, such as all
those pesky `:init` routines that need to execute before user `:main` does
is also a performance win. Of course, this is only a win for programs which
use the Parrot frontend or maybe other embedding applications (such as
`pbc_to_exe` fakecutables) which borrow similar ideas.

In the `whiteknight/frontend_parrot2` branch on Github, I've been working
towards exactly this kind of situation. I've created a new frontend which
attempts to bootstrap into a PIR entry-way program as early as possible. This
PIR entry program, which I've been calling `prt0.pir` tries to do as much
processing of command-line arguments as possible, including loading of PBC
files and compilation of PIR and PASM input files, and other details. In the
process I am trying to minimize the amount of C code in
`frontend/parrot2/main.c` and hopefully bring some performance improvements
along for the ride. I haven't done any benchmarking yet because I don't have
all the details in place yet. One thing that I am currently missing is linking
`prt0.pbc` into the `parrot` executable. Instead, I am currently loading it in
at runtime from a separate file, which brings unnecessary overhead. My hacking
goals for tonight and tomorrow are to get this issue resolved and start with
some serious benchmarking to see if my hypothesis plays out.

Over the weekend I made the switchover in my branch so that the `parrot`
executable is built from the new `frontend/parrot2/main.c` file instead of
the old `frontend/parrot/main.c`. Miniparrot, the bootstrapping step which is
used to compile the config hash, still is built from the old file and is now
used to also compile `prt0.pir`. I see this as being a perfectly acceptable
build process, and a great additional use for miniparrot. Once I made this
switch and a few small tweaks, I was pleased to see that the build completed,
the tests suite ran, and most tests even passed. Of the tests that fail, the
majority of them seem to be tests related to backtraces, where a new
"`__PARROT_MAIN_ENTRY__`" function, the `:main` function in `prt0.pir`, is
now appended at the top of all backtraces. One final piece of functionality is
to find a way to remove that entry, so backtraces continue to look the way
they always have looked.

One big change that I did have to make for this new setup is in argument
processing. Previously, a `:main` routine was expected to take a single
paramter: An array of command-line strings. For the new frontend, it's much
faster to separate out arguments in C using fast pointer arithmetic, and do
processing on lists which have already been sorted in PIR. The new
`__PARROT_ENTRY_MAIN__` routine takes two parameters. The first is a string
array of "system arguments", things that affect the behavior of Parrot but
which are not passed to user code. The second is the set of arguments which
are passed to the user code. Here's how the parrot commandline looks:

    ./parrot <sys_args> my_file.pir <user_args>

In the C frontend, we break the arguments up into 4 basic categories:
1. Arguments which affect interpreter creation, and therefore need to be
   parsed out *before* the interpreter is created.
2. Arguments which are processed in C code, but are processed after the
   interpreter is created.
3. Arguments which are for system-related stuff, but which can be processed
   from the PIR entry.
4. Arguments which are supposed to be passed to user code

My goal in this branch is to move as many arguments from category 2 to
category 3 as I can. Anything that works can certainly stay where it is, and
some of these changes can be made after the initial branch is merged.

To get this branch into mergable shape, which probably won't happen before
3.7 considering the magnitude of the changes involved, I have a few tasks to
finish up with: I need to fix the remaining test failures, fix loading of
`prt0.pbc` into parrot, do some benchmarking to see if it is indeed an
improvement (or if it can be made better with some tweaks), and then update
the `pbc_to_exe` tool to use a similar mechanism. Once all that is done, and
we've tested the hell out of it, we can talk about merging this branch to
master.

Once merged, the `:init` flag will no longer be semantically different from
the new `:tag("init")` syntax, when parrot is used from the commandline
executable or from a pbc_to_exe fakecutable. That's a very important step in
the deprecation of the former, and is going to enable us to clean up a
pretty big chunk of dirty code in IMCC and the packfile subsystem.



