---
layout: post
categories: [Parrot, GC]
title: Destructors are Hard
---

<<<<<<< HEAD:drafts/gc_destructors.md
In my last post I mentioned some of the work I was trying to do with GC
finalization and destructors. I promised I would publish a longer and more
in-depth post about destructors, what the current state is, what I am doing,
and what still needs to be done.
=======
In [my last post](/2012/05/20/pending_branchwork.html) I mentioned some
work involving the GC, finalization and destructors. Today I'm going to
expand on some of those ideas, talk about what the current state of
destruction and finalization are in Parrot, some of the problems we have with
coming up with better solutions, and some of the things I and others are
working on to get this all working as our users expect us to. I apologize
in advance for such a long post, there's a lot of information to share, and
hopefully a much larger architectural discussion to be started.
>>>>>>> 3d0a502723cb7124eb717d7e82bac5ecc567ac31:_posts/2012-05-23-destructors_are_hard.md

Destructors are hard. The idea behind a destructor is a simple one: We want to
have a piece of code that is guaranteed to execute when the associated object
is freed. Memory allocated on the heap is going to get reclaimed en masse by
the operating system when the process exits. However, things such as handles,
connections, tokens, mutexes, and other remote resources might not necessarily
get freed or handled correctly if the process just exits, or if the object is
destroyed without some sort of finalization logic performed on it. Here's a
sort of example that's been bandied about a lot recently:

    function main () {
        var f = new 'FileHandle';
        f.open("foo.txt", "w");
        f.print("hello world");
        exit(1);
    }

In this example we would expect that the text `"hello world"` would be written
to the `foo.txt` file. However, because the text to be written may be buffered
(both in Parrot and by the OS), there's a very real chance that the data won't
get written if we do not call the finalizer for the `FileHandle` PMC.

Obviously, the brain-dead solution to this particular problem is to manually
close or flush the file handle:

    function main () {
        var f = new 'FileHandle';
        f.open("foo.txt", "w");
        f.print("hello world");
        f.close();
        exit(1);
    }

However, the whole point of having things like finalizers ("destructors") and
GC is to make it so that the programmer does not need to worry about little
details like these. The program should be smart enough to find dead objects
in a timely manner and free their resources. Beyond that, many programming
languages (with special emphasis on Perl6) require the availability of
reliable and sane destructors.

In the remainder of this post I would like to talk about why destructors are
hard to implement correctly, why Parrot does not currently (really) have them,
and some of the ideas we've been kicking around about how to add them.

First, let's cover where we currently stand. Parrot does have destructors, of
a sort, in the form of the `destroy` vtable. That routine is called by the GC
when the object is being reclaimed, during the sweep pass. A side-effect of
this implementation is that if PMC `A` refers to PMC `B` and both are being
collected, it's very possible that `A`'s destructor tries to access some
information in `B` *after `B` has already been reclaimed*. Think about a
database connection object that maintains a socket on one side, and a hash of
connection information on the other. The socket probably cannot perform a
simple disconnect, but instead should send some sort of sign-off message first
to alert the server that it can proceed with its own cleanup. The socket PMC
would need information from the connection information hash to send this final
message, but if the hash had already been reclaimed the access would fail with
undefined results.

This situation has lead to more than a few calls for ordered destruction. In
one of the most common and severe cases, Parrot's Scheduler PMC was being
relied upon by various managed PMCs. When a Task PMC was destroyed, at least
in earlier iterations of the system, it would attempt to send a message to the
Scheduler that it was no longer available to be scheduled. Ignore for a moment
the fact that the Task could not possibly have been reclaimed in the first
place if the Scheduler had a live reference to it, and if the Scheduler was
still alive itself.

Because of some of these order-of-destruction bugs, GC finalization (a final,
all-encompassing GC sweep path guaranteed to execute all remaining destructors
prior to program exit) had been turned off. That and performance reasons.
Turning off GC finalization leads to the problem above where data written to
the FileHandle is not not flushed before program exit. You are probably now
starting to understand the bigger picture here.

Having ordered destruction means essentially that we should be able to have an
acyclic dependency graph of all objects in the system with destructors.
However, maintaining this in the general case is impossible and attempting to
approximate it would be very expensive in terms of performance. In any case,
this is just a way to work around the problem of our naive sweep algorithm,
which destroys and frees dead objects in a single pass, and not a real
solution to the larger problems. A far better idea, recently suggested by
hacker Moritz, is a 2-pass GC sweep.

In the 2-pass case the GC sweep phase would have two loops: the first to
identify all PMCs to be swept (from a linear scan of the entire memory pool),
execute destructors on them and add them all to a list, and the second to
iterate over that list (after all destructors had been called) and reclaim the
memory. Because of the linked-list setup of the GC, this second pass could,
conveivably, be almost free because we could simply append this list of swept
items to the end of the free list for an `O(1)` operation , and the first pass
would be no less friendly on the processor data cache than our current sweep
would be. This, in theory, solves our problem with ordered destruction, and
should allow us to re-enable GC finalization globally without having to worry
about these kinds of bugs causing segfaults in the final milliseconds of a
program.

So that's the basics of our current system and our problem with GC
finalization, and shows us how we would proceed to make sure destructors were
always called as a guarantee of the VM. However, this doesn't begin to address
any of the problems with destructors that will plague their implementation and
improvement. I'll talk about that second subject now.

Destructors, as I said earlier, are hard. In the case of GC finalization,
after the user program has executed and exited, it's relatively easy to loop
over all objects and call destructors. It is those destructors which happen
during normal program execution that cause problems.

In the C++ language, destructors have certain caveats and limitations. For
instance, we can't really throw exceptions from destructors, because that may
crash the program. Not just an "oops, here's an exception for you to handle",
but instead a full-on crash. In Parrot we can probably be smarter about
avoiding a crash but not by much. It's a limitation of the entire destructors
paradigm.  Let me demonstrate what I'm talking about.

Let's say I have this program, which opens up a filehandle to write a message
and then starts doing something unrelated to the filehandle but expensive with
GC:

    function foo() {
        var f = new 'FileHandle';
        f.open("foo.txt", "w");
        f.write("hello world!");
        f = null;       // No more references to f!

        for (int j = 0; j < 1000000; j++) {
            var x = new MyObject(j);
            x.DoSomething();
            x.DoSomethingElse();
            x.DoOneLastThing();
        }
    }

Somewhere along the line, when the GC runs out of space, it's going to attempt
a sweep and that means that `f` is going to be identified as unreferenced,
finalized and reclaimed. The question is, where? The thing is that we don't
know where GC is going to run for a few reasons:

1. We don't know how many free headers GC has left in the free list before it
   has to sweep to find more.
2. We don't know how many PMCs are being allocated per loop iteration, because
   the various methods on `x` could be allocating them internally, and all PCC
   calls currently generate at least one PMC, and this is a lot of pressure.

So at any point in that loop, any point at all, GC could execute and reclaim
the FileHandle `f`. That calls the destructor, flushes the output, and frees
the handle back to the operating system. Good, right? What if there is a
problem closing that handle, and the destructor for FileHandle tries to throw
an exception (or, if this example isn't stoking your imagination, imagine that
`f` is an object of type `MyFileHandleSubclass` with an error-prone
finalizer).

There are a few options for what to do here. The first option is that we throw
the exception like normal. This means that the loop code with the `MyObject`
variables, which is running perfectly fine and has no reason to throw an
exception by itself, is interrupted in mid loop. The backtrace, if we provide
one at all, probably points to `MyObject` but with an exception type and
exception message indicative of a failed FileHandle closing. Initial review by
the poor developer doing the debugging will show that there are no filehandles
trying to close inside this loop and then we get a bug report because a
snippet of code which is running just fine exits abruptly with an error
condition which it did not cause. The solution for this, wrapping every single
line of code you ever write in exception handlers to catch the various
possible exceptions thrown from GC finalizers, is untenable from a developer
perspective.

A second option is that we somehow disallow things like exceptions from being
thrown from destructors, because there's no real way to catch them rationally.
This seems reasonable, until we start digging into details. How do we disallow
these, by technical or cultural means? And if we're relying on cultural means
(a line in a .html document somewhere that says "don't do that, and we won't
be responsible if you do!"), what happens if a hapless young programmer does
it anyway without having first read all million pages of our hypothetical
documentation? Does Parrot just crash? Does it enter into some kind of crazy
undefined state? Obviously we would need some kind of technical mechanism to
prevent bad things from happening in a destructor, though the list of
potentially bad things is quite large indeed (throwing exceptions, allocating
new PMCs, installing references to dead about-to-be-swept objects into living
global PMCs, etc) and filtering these out by technical means would be both
difficult and taxing on performance. When you consider that even basic error
reporting operations at an HLL level, depending on syntax and object model
used, may cause a string to be boxed into a PMC, or a method to be called
requiring allocation of a PMC for the PCC logic, or whatever, we end up with
finalizers which are effectively useless.

A third option is that we could just ignore certain requests in finalizers,
such as throwing exceptions. If an exception is thrown at any point we just
pack up shop, exit the finalizer and pretend it never happened. This works
fine for exceptions, but does nothing for the problem of a finalizer
attempting to store a reference to the dieing object into a living object. I
don't know why a programmer would ever want to do that, but if it's possible
you can be damned sure it will happen eventually. Also, when I say "pack up
shop", we're probably talking about a `setjmp`/`longjump` call sequence, which
isn't free to do.

The general consensus among developers is that errors caused by programs
running on top of Parrot should never segfault. If you're running bytecode in
a managed environment, the worst that you should ever be able to get is an
exception. Segmentation faults should be impossible to get from a pure-pbc
example program.

However, as soon as you introduce destructors, suddenly these things become
possible. And not just from specifically malicious code, even moderately
naive code will be able to segfault by storing a reference to a dieing PMC
in a place accessible from live PMCs. Unless, that is, we try to do something
expensive like filtering or sandboxing, which would absolutely kill
performance.

And this point I keep bringing up about dead objects installing references to
themselves in living objects is not trivial. Our whole system is built around
the premise that objects which are referenced are alive and objects which are
no longer referenced can be reclaimed by GC. Throughout most of the system we
dereference pointers as if they point to valid memory locations or to live
PMCs. If we turn that assumption around and say that dead objects may still be
referenced by the system, then we lose almost all of the benefits that our
mark and sweep GC has to offer. Specifically we would either have to install
tests for "liveness" around *every single PMC pointer access*, which would
bring performance to a standstill. Otherwise, we need to have a policy that
says the user at the PIR level is able to create segfaults without
restriction, though officially we declare it to be a bad idea. It's not
just a matter of having to test PMCs to make sure they are alive, the memory
could be reclaimed and used for some other purpose entirely! Meerly accessing
a reclaimed PMC could cause problems (segfaults, etc) or, if the PMC has
already been recycled into something like a transparent proxy for a network
resource, send network requests to do things that you don't want to have
happen! The security implications are troubling *at best*.

The only real solution I can come up with to this problem, and it's not a very
good one, is to add a "purgatory" section to the GC, where we put PMCs during
GC sweep but we do not actually free them. The next time GC runs, anything
which is still in purgatory is clearly not referenced and can be freed
immediately. Anything that is no longer in purgatory has been "resurrected"
by some shenanigans and has to be treated as still being alive *even though
its destructor has already been called*. In other words, we take a performance
hit and enable zombification in order to prevent segfaults. I don't know what
we want to do here, this is probably the kind of decision best left to the
architect (or tech-savvy clergy) but I just want to point out that none of our
options are great.

I've also brought up the problem with allocating new objects during a
finalizer. Why is this a problem? Keep in mind that GC tends to execute when
we've attempted to allocate an object and have none in the free list. If we
have no available headers on the free list, are already in the middle of a
GC sweep and ask to allocate a new header, what do we do? Maybe we say that
we invoke GC when we have only 10 items left (instead of 0) on the free list,
guaranteeing that we always have a small number of headers available for
finalization, though no matter what we set this limit at it's possible we
could exhaust the supply if we have many objects to finalize with complex
finalizers. Every time a finalizer calls a method or boxes a string, or
does any of a million other benign-sounding things PMCs get allocated. If we
try to allocate a PMC when there are no PMCs on the free list and we're
already in the middle of GC sweep, the system may trigger another recursive
GC run.

Another option is that we could maintain multiple pools and only sweep one at
a time. If one pool is being swept we could allocate PMCs from the next pool
(possibly triggering a GC sweep in that second pool and needing to recurse
into a second pool, etc). Maybe we allocate headers directly from malloc
while we're in a finalizer, keep them in a list and free them immediately
after the finalizer exits. We have some options here, but this is still a very
<<<<<<< HEAD:drafts/gc_destructors.md
real problem that requires very careful consideration. Something like a
semi-space GC algorithm might help here, because we could allocate from the
"dead space" before that space was freed.

Or we could try to immediately free some PMCs during the first sweep pass, and
use those headers as the free list from which to allocate during destructors.
This raises some problems because it would be very difficult to identify PMCs
which could be freed during the first pass without negating any references
which are going to be accessed during the destructors. Also, we run into the
(rare) occurance where all the PMCs swept during a particular GC run have
destructors, and there are no "unused" headers to immediately free and
recycle for destructors.
=======
real problem that requires very careful consideration. Again, I don't have an
answer here, just a long list of terrible options that need to be sorted
according to the "lesser of all evils" algorithm.
>>>>>>> 3d0a502723cb7124eb717d7e82bac5ecc567ac31:_posts/2012-05-23-destructors_are_hard.md

Let's look at destructors from another angle. Obviously a garbage-collected
system is supposed to free the programmer up from having to manually manage
memory (at least) and possibly other resources as well. You make a mess and
don't want to clean it yourself, the GC comes along after you and takes
care of the things you don't wnat to do yourself. On one hand the argument
can be made that if you really care about a resource being
cleaned in a responsible, timely manner, that you call an explicit finalizer
yourself and leaving those kinds of tasks to the finalizer is akin to saying
"I don't care about that object and whatever happens, happens." After all, if
you can't throw an exception from a destructor and if the destructor is called
outside normal program flow with no opportunity to report back even the
simplest of success/failure conditions, it really doesn't matter from the
standpoint of the programmer whether it succeeded or silently failed. Further,
if the resource is sensitive, you don't clean it explicitly and Parrot later
crashes and segfaults because some uninformed user created a zombie PMC
reference, your destructor cannot and will not get called no matter what. If
all sorts of things at multiple levels can go wrong and prevent your
destructor from running, does it *really* matter if the destructor gets called
at all?

Another viewpoint is that destructors don't need to be black-boxes, and we
don't care if they have problems so long as they've given a best effort to
<<<<<<< HEAD:drafts/gc_destructors.md
free the resources, those efforts have a decent expected chance of success,
and they have an opportunity to log problems in case somebody has a few
moments to spare reading through log files. After all, if a FileHandle fails
to close in an automatically-invoked destructor, it also would have failed to
close in a manually-invoked one and what are you going to do about it? If the
thing won't close, it won't close. You can either log the failure and keep
going with your program (like our destructor would have done automatically) or
you can raise hell and possibly terminate the program (like what *could*
happen if an exception is thrown from a destructor). In other words, when
you're talking about failures related to basic resources at the OS level,
there aren't many good options when you're writing programs at the Parrot
level.

I suspect that what we are going to end up with is a system where we
allocate a temporary managed pool of PMCs to be available, and allocate all
PMCs during a destructor from that pool. After GC, we clear the emergency
pool at once. This solution adds a certain amount of complexity to the GC and
also does nothing to deal with the zombie references problem I've mentioned
several times. We'd have to make a stipulation that PMCs allocated during
a destructor *may not* themselves have automatic destructors.

Things start to get a little bit complicated no matter what path we choose.
This is the kind of issue where we're going to need lots more input,
especially from our users.
=======
free the resources and they have an opportunity to log problems in case
somebody has a few moments to spare reading through log files. After all, if
a FileHandle fails to close in an automatically-invoked destructor, it also
would have failed to close in a manually-invoked one and what are you going
to do about it? If the thing won't close, it won't close. You can either
log the failure and keep going with your program (like our destructor would
have done automatically) or you can raise hell and possibly terminate the
program (like what *could* happen if an exception is thrown from a
destructor). In other words, when you're talking about failures related to
basic resources at the OS level, there aren't many good options when you're
writing programs at the Parrot level. If you're not so hot at OS
administration, there might not be anything you can do no matter what.

In Parrot we really want to enable PMC destruction and GC finalization.
As things stand now you can run `destroy` vtables written in C, usually
without issue. However when we expose this functionality to the user we are
talking about executing PBC, in a nested runloop (at least one!), with fresh
allocations and all the capabilities of PBC at your disposal. As soon as you
open that can of worms, the many problems and problematic possibilities become
manifest. The security concerns become real. The performance implications
become real. I'm not saying that these are problems we can't solve, I'm only
pointing out that they haven't been solved already because they are hard
problems with real trade-offs and some tough (and unpopular) decisions to be
made.
>>>>>>> 3d0a502723cb7124eb717d7e82bac5ecc567ac31:_posts/2012-05-23-destructors_are_hard.md
