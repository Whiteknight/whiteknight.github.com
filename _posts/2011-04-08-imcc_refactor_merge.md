---
layout: post
categories: [Parrot, IMCC]
title: IMCC Refactor Merged!
---

Last night I finally reached a milestone that I was starting to worry might
never come. Last night, I finally merged in the `whiteknight/imcc_compreg_pmc`
branch into Parrot master. This branch shouldn't bring too many obvious
changes to the average parrot user, but it does do a hell of a lot of
necessary infrastructure cleanup which has been sorely needed for some time.
It is the first step in a long path which will eventually lead to IMCC, the
current standard PIR/PASM assembler, being separated completely from libparrot
and being offered as a separate (and optional!) component.

In the rest of this post I will talk about what the old system looked like and
what some of the biggest problems were, and I will also talk about the various
things that were changed in this branch. The complete
[branch merge diff][diff] for this work is pretty gigantic. I don't recommend
you try loading it in its entirety unless your computer has a lot of available
RAM or unless you really don't like github and want to suck up a bunch of
their bandwidth.

[diff]: https://github.com/parrot/parrot/commit/1842a6ef65a33c24c6f7e21004f006ce1e9e5b3d

## The Old Way

A big part of the problem with IMCC, which I've mentioned before, is the fact
that it was built in to parrot inseparably. There was no encapsulation to
speak of whatsoever. IMCC poked directly into the internals of Parrot, in an
intrusive way which most other subsystems don't do. In response, Parrot
expected results from IMCC to be loaded directly into the interpreter without
going through any kind of a standard interface. Because it wasn't needed, such
an interface didn't even exist, which means other compilers wouldn't have any
hope of loading in new bytecode files without using IMCC as an intermediary.

The results of this situation were manifest and easy to see. Fixing even the
most simple of bugs related to IMCC and packfiles would often become a huge
nightmare because various bits of code were too highly coupled and
intertwined. Fixing or modifying things in IMCC would create
difficult-to-debug problems in other parts of Parrot.

Since IMCC relies on a secret, undefined, "hidden" interface to the rest of
Parrot, it becomes impossible to replace IMCC. You could not use Parrot
without IMCC. You could not disable it. You could not perform even basic
behaviors in Parrot without going through IMCC at some point. This creates an
environment where, in addition to the problems I've already mentioned and
those I will mention hereafter, where other compilers cannot be used except as
a frontend to feed PIR or PBC code into IMCC.

Command-line arguments? Half of them were parsed (or, re-parsed) inside IMCC,
and caused flags to be set on the IMCC info structure (`imc_info_t`) which
other parts of Parrot would need to reference.

Loading bytecode? All input files went in to IMCC, which then differentiated
between PIR, PASM, and PBC files by doing comparisons on the file extension
string.

Executing the `:load`, `:init`, or `:main` subs in your file? Those things
were all detected and invoked by IMCC. This also created a situation where
subs were hard to execute without IMCC, since much of the setup work to
prepare a new interpreter runloop happened inside IMCC.

All of this ignores the point that IMCC is an extremely messy clump of code.
It's origins were humble and even admirable, but by the year 2011 it has
become a mismash of hackjobs where people were repeadedly adding "one more
quick fix" to implement all sorts of new features and behaviors. We've
identified it as a problem long ago and have been wanting to replace it, but
because it is so intertwined into Parrot we have not been able to excise it
before now.

In short, the situation was extremely messy. IMCC really needs to be separated
from libparrot, but there was a huge web of dependency issues that needed to be sorted out first. This is where the `imcc_compreg_pmc` branch and its
various ancestor branches come in.

## The New Way

### Basic Cleanup

The public interface of IMCC, if one can be so charitable as to call such a
mismash an "interface" was a huge stumbling block to any refactor efforts.
There were [huge swaths of duplicated code shared][old_interface] between
several similarly-but-deceptively named interface routines, and no clear
picture about how a compilation was actually performed in a general way. Any
nontrivial refactor of IMCC would have to do one of two things: Either attempt
to modify all interface functions and all duplicate code (including the few
copies that were inexplicably hidden in weird places) so that they all
continued to work with the refactors, or else clean up and unify all public
interfaces into a single call-in point that all users could rely on. I choose
to do the second option.

[old_interface]: /2011/01/18/imcc_interface_functions.html

My first task in working with IMCC was to fix the public interface for it.
In this case I am using the term "public interface" to refer to those routines
which IMCC provides to be called by libparrot. This is not the same interface
as would be needed in an embedding application which needed to call into IMCC
directly without routing the request through libparrot first. That's a
separate kind of interface and I will discuss it later.

Without modifying any of the actual logic, I went through painstaking effort
to move all related interface functions into a single file, and then slowly
refactored common bits of each of them out into various helper functions. I
was able to merge all interfaces into a single shared code path, which two
thin wrapper functions over it to handle specific use-cases. With this, the
system was finally ready to accept a larger set of refactors that I had
planned.

### The `imc_info_t` Structure

In old IMCC, almost every function took a `PARROT_INTERP` as the first
argument. From there, we could get a reference to the `imc_info_t` structure
which holds the running state of IMCC by using the `IMCC_INFO(interp)`
macro. This was less than ideal, and created a weird circular dependency:
Almost every IMCC function required a valid Parrot interpreter structure to
operate, and a complete initialized interpreter contained an `imc_info_t`
structure (which, coincidentally, more components than just IMCC would expect
to find).

One of the first tasks I undertook was to break this dependency. Parrot's
interpreter no longer creates an `imc_info_t` structure when the interpreter
is initialized, and now no longer even has a pointer to it in the interpreter
structure itself. All functions in IMCC now take an explicit `imc_info_t`
structure as the first argument, and `imc_info_t` now contains a pointer to
the "parent" or "owner" interp object which created it.

This first part of the refactor was both the most time-consuming but the least
thought-provoking. Much of the work could be done by running simple sed
scripts to automatically replace `PARROT_INTERP` with `imc_info_t *imcc` and
replace `IMCC_INFO(interp)` with `imcc`. What was good about this process was
that it forced me to read through *every single line of code* in every file
in IMCC to verify that the translation worked as expected and to perform some
manual touch-ups. This was both enlightening and depressing. I now have a
much better understanding of the original design and even the original genius
of the first version of IMCC. It really was, at one point, a great assembler
program and I can see why early Parroteers were so eager to adopt it into
libparrot wholesale. At the same time, I can clearly see the cobbled-together
nature of it. The differences in coding style between the original coder and
the subsequent patchers, hackers, and weary developers is as obvious as night
and day. In some cases, I suspect the newer additions solved problems. In
many cases the reverse is true. In almost all cases, the clearness of purpose
and vision was distorted and lost.

With the `imc_info_t` structure pulled out of the interpreter and put into
a better relationship with it, I was then presented with a new problem: Where
do I hold the `imc_info_t` structure pointer so that it can be accessible from
libparrot without creating a new dependency?

### The IMCCompiler PMC

Parrot defines a compiler registration mechanism called "`compreg`". A compreg
is a compiler object registered with the interpreter by name. Registration and
fetching compilers are both done through various forms of the `compreg`
opcode.

IMCC provides two registered compilers: one for PIR, and one for PASM. Here's
how they would have been used in code:

    $P0 = compreg "PASM"    # Get the PASM compreg
    $P1 = $P0(pasm_code)    # Compile a code string into an invokable
    $P1()                   # Execute the compiled code

    $P2 = compreg "PIR"     # Get the PIR compreg
    $P3 = $P2(pir_code)     # Compile a string into an invokable
    $P3()                   # Execute the compiled code

Notice that this mechanism is distinctly different from the compiler interface
described in the draft PDD31. Most other compilers would provide a
`.compile_file()` method, or related methods to perform various activities.
Not so with IMCC.

The PMCs registered as the PIR and PASM compiler objects were simple NCI
PMCs: basic wrappers around a C-level function which performed basis
translating of arguments but nothing else fancy. NCI PMCs could not be
reliably or flexibly used to maintain state for instance, or to implicitly
inject a saved parameter into the call sequence. So, even if we did find a way
to store the `imc_info_t` structure with the NCI PMC itself, there's no real
way to tell the NCI PMC to automatically insert that as the first argument
to all invokations. And even if we did come up with some kind of weird hack
to allow this, it *still* wouldn't be right, because it *still* wouldn't be
following the PDD31 draft or the good example set by the HLLCompiler class in
PCT.

With the `imc_info_t` structure separated from the Parrot interpreter, I
needed to create a real compiler object to use with it. For this purpose, I
added a new IMCCompiler PMC type. The IMCCompiler PMC is a wrapper object
around IMCC's new public interface. At the moment it is a built-in PMC type,
but in the near-term future it will instead become a dynpmc to only be loaded
on demand when necessary.

What's good about the new IMCCompiler PMC type is that it can be used exactly
the same way as the old NCI PMCs could be used. However, it also adds in some
new methods to conform with PDD31 to help provide a migration path to using
the better interface.

Now that I had a new PMC type to encapsulate all IMCC state information and
provide a nice interface for it, I had to find a way to create that PMC. If
we created it inside libparrot during interpreter initialization, that would
have defeated the purpose. It would have been like putting a spit-shine on
a turd. What we really needed to do was create the new IMCCompiler PMC
*outside* libparrot, and register it as a normal compiler just like any other
compiler object. The natural place to do this was in the `parrot.exe`
frontend code. The parrot executable is a thin wrapper around libparrot which
turns a string of commandline arguments into a sequence of instructions for
creating and executing a Parrot program. By creating and registering the
IMCCompiler PMC from the frontend, I could ensure that it would be separate
from libparrot and that alternate frontends could be created without any
dependency on IMCC at all!

However, there is one snag in the road: IMCC throws exceptions. This means
that any IMCC operations called from an embedding application needed to go
through an exception-catching mechanism like is used by Parrot's embedding
API. Again, I was presented with two options: I could either create some
call-in routines in Parrots embedding API to call in to IMCC, or I could
create a new embedding API for IMCC to use independent of the Parrot API.
Again, I chose the second option.

### IMCC Embedding API

IMCC's old public functions were mostly in `compilers/imcc/main.c` (some were
inexplicably hidden in other files). I've kept this file as the central point
for main-entry routines and interfaces expected to be called into from inside
Parrot's execution loop (such as the IMCCompiler PMC). However, I had to add
a new file for an external API. I added `compilers/imcc/api.c` for that
purpose.

The new IMCC embedding API is modeled strongly after the Parrot embedding API,
although is not identical. It uses a different naming convention and has
some slightly different setup and teardown logic. Other than the minutia, the
concepts are all the same. Every function returns a boolean to signal success,
error information can be queried from the appropriate parrot API routines, and
all routines return all values through pointers in the parameter lists. It's
a pattern that I spent a lot of time designing and I cam convinced that it is
the best solution to this particular set of problems.

With the new embedding API in place, I was able to rewrite the parrot
executable frontend to use the new API, to create and register IMCCompiler
PMCs (one for "PIR" and one for "PASM"). IMCC could be created externally to
libparrot and in some cases libparrot could be used without creating these
compiler objects at all. This was getting to a great point, but there were a
few more details left to cover. The first is something I found while
implementing the new API: IMCC handled errors in several incompatible ways.
With the new API automatically catching exceptions and communicating them
in a standard way to the embedding code, we needed a standard way to report
errors in IMCC to the outside world. In several cases IMCC would not throw
an exception but would instead return a boolean status result. Calls to the
new embedding API would return from these situations with a "success" flag,
even though the underlying compilation failed. To fix this problem I had to
change the way IMCC handles errors.

### IMCC Error Handling

IMCC had two different mechanisms for handling errors. It used these two
mechanisms interchangably without rhyme or reason. The first was to create and
throw a standard Parrot exception. This is a good thing, and allows parrot
to catch and handle the error, provide backtraces to the user, recover from
errors, etc. In short, we want to use Parrot's exception system for errors.
The other way was a homebrew "exception-alike" system in IMCC which used a
jump buffer to jump to the call-in point and return a boolean flag for
failure. These errors typically contained much less diagnostics information
and were harder to detect and deal with from outside IMCC, especially when
you consider the range of consumers who could be using it.

With no small sense of happiness and enthusiasm, I ripped out the old IMCC
error handling mechanism and replaced it with standard Parrot exceptions.

[At this point][first_attempt] I thought I was finished. I could build, run,
and test Parrot with success. So I sent out an email asking for testing from
other developers as well in preparation for a merge.

[first_attempt]: /2011/03/16/imcc_current_status.html

No dice. We started getting back errors from some people, especially kid51
who was on a memory-contained test machine. It turns out, the GC was quite
angry at my branch.

### PackFile PMCs

IMCC is not GC safe. Never has been, maybe it never will be. The problem is
that it creates a large number of PMCs which don't always get anchored
immediately, and might get prematurely collected if Parrot's GC runs during
compilation. To avoid this, we disable GC when we call into IMCC, and
reenable it again when IMCC exits. With the old error handling system this
was easy enough, because IMCC would return a boolean to signal success or
failure. With the new exceptions-based system we run into a problem because
IMCC doesn't necessarily *exit*. Sometimes it aborted with an unhandled
exception and jumped immediate to some handler elsewhere in the program.

And on top of that there was another issue. IMCC was returning a `PackFile*`
structure, not any kind of PackFile PMC. These PackFile structures contained
a number of other PMCs, such as the Sub PMCs in the program and the various
constant PMCs in the constants table. If the `PackFile` structure doesn't get
marked for GC, all those PMCs it contains disappear. This is bad. I refactored
IMCC and added a few interface methods to the packfile subsystem API to create
and return a new PMC type which automatically marks the `PackFile` and all
its contents. This solved the last of the problems and again I opened the
branch up for widespread testing.

While everything wasn't perfect, we were close enough to merge. Last night
I finally did it.

## Aftermath

As my description of the development process above should show, this was not
an easy nor a straight-forward task. IMCC was so messy and so coupled with
Parrot that fixing one problem created a cascade of additional problems.
Typically I like to make small changes and merge them in atomically, but with
this work I had to do everything all at once. There were no periods of
stability in the middle of this refactor where things worked well enough to
merge bits back into master. From the moment I started it basically become
all or nothing.

At some points in the process, I was worried that the answer might be
"nothing". There were a few points in there when I was worried that we had
hit bugs so big and devastating that I would not be able to continue. Luckily
with some hard work and some valuable help and input from other Parrot hackers
I was able to get through the wost of them.

### Current Status and Future Plans

So where are we now? The code merged in to master last night, but what has
really changed? From the perspective of the average user, the answer is
"nothing". IMCC still exists, and is still created as part of the default
Parrot frontend. It's now possible to create a frontend which does not
create this compiler, or create a frontend which registers an alternate PMC
as the PIR compiler, but this is probably not a capability that many people
will be using right now.

Performance and memory consumption should not change much.

What is different now is that IMCC is not necessary. It's not a necessary part
of the code, it isn't a necessary stage in interpreter initialization, and
it isn't required to be available. However, it is still built in as part of
libparrot, and even if you don't use it the code is still there.

So what are the next stages in this work? I don't have any immediate plans to
work on this myself, but if some other intrepid coder wants to carry the
torch forward, these are the steps that I think we can take from here:

1. Modify the makefile to build IMCC as a separate library. Call it something
   like "libparrot-imcc".
2. Once it's a separate library, we should be able to create alternate
   frontends which do not rely on it. One such possible fronend could be for
   PIRATE, a new PIR compiler. Instead of IMCC, the new frontend would load
   and register PIRATE as the default PIR compiler.
3. Move IMCC out of the parrot repo into it's own repo. We can still make it
   readily available through a git submodule or something, but it should live
   in it's own place and not be in the Parrot repository by default.

Once that is done, the mission is complete. The first part could probably be
done in a short weekend, and would arguably have the most benefit. After
the two libraries are separated, the sky is really the limit.
