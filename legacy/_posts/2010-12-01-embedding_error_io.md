---
layout: post
categories: [Parrot, Embedding]
title: Embedding API Error IO
---

The program performs an illegal operation and cannot continue it's current
course of execution. It packs up the relevant information--location,
backtrace, type, an error code, an error message, severity, and other
details--and throws it as an exception. Parrot's concurrency scheduler scans
through the list of installed exception handlers but cannot find a suitable
candidate to handle it. Crap. Panic.

In this situation libparrot calls the routine `die_from_exception`, the
action of last resort to stop the actions of the interpreter and pass control
back to the embedding application. The application needs all the necessary
information from that exception object so it can optionally display it to the
user or use it to decide upon a new course of action for itself.

It's worth asking ourselves what state the interpreter is in immediately
following an unhandled exception. Currently, the interpreter calls the
function I mentined above, `die_from_exception`. This function prints out a
bunch of information to `stderr`: The text of the error message from the
Exception object and a backtrace from the location where the exception was
thrown. After printing out this information and a little bookkeeping, it calls
`Parrot_exit` to run any exit handler routines and finally call the libc
`exit()` function to close the program.

That's right, libparrot calls `exit()`, it never passes control back to the
embedding application.

There are several things wrong here, and I'll discuss them all in a bit. It's
most important right now to talk about the state of the interpreter at this
point. Once we call `Parrot_exit`, the exit handlers have fired and been
freed. `Parrot_exit` also shuts down the GC, so we can no longer reliably
create new PMCs or STRINGS. If we attemp to re-enter the interpreter it
won't have an active GC, which will cause memory use to explode.

Once we call `Parrot_exit`, at least as it exists in trunk, we are
completely done with the interpreter and we cannot reliably create or use any
PMC or STRING objects, or call any other Parrot subroutines or methods again.
Since control flow never passes back to the embedding application, we can't
do anything else, either. Even if we did jump back to the embedding
application, we couldn't use PMCs or STRINGs to communicate error information
back to it because the interpreter is dead. This is all extremely bad. So what
do we do instead?

First and foremost, libparrot should rarely--if ever--print anything to
`stderr` directly. This, if it happens at all, is the option of last resort.
 Output formatting and the output vector should be the sole
providence of the embedding application. The first change we need to make is
to take all the necessary textual information and either package it up in a
form that the embedding application can use, or flip an error flag and provide
a series of API calls that the embedding API can use to get the information it
needs. A PMC seems like the natural choice for this, since our Exception
objects would already contain all the information we need: The location
information and the ability to
generate a backtrace, the type and severity information, the human-readable
message string, etc. So, it seems like this is what we really want to use. A
second-place offering would be something like a StringBuilder PMC, which we
could use to construct all the output as we would currently show on stderr,
but instead prepare it and dump it into a STRING for use by the embedding
application. It's not ideal to package the backtrace up with the message, but
if that were our only option it's what we would have to do.

I don't think it's what we have to do.

The reality is that the Parrot interpreter can probably be reused after
having exited so long as we don't finalize the GC and we don't run all the
exit handlers. Basically, so long as we don't call `Parrot_exit` (at least
as it is implemented in master now) we can call back into the interpreter
later. We need to create a new function, probably `Parrot_jump_out` or
something, and use that most of the time when we are currently calling
`Parrot_exit`. Then, we only call `Parrot_exit` when we are actually closing
the interpreter down for good, probably during (or immediately after) the
call to `Parrot_destroy` (which kills the interp and frees most of it's
resources).

Once we have the jump-out semantics working correctly, we will be able to get
out of the realm of the interpreter without destroying/disabling the GC, which
means we can pass the raw Exception objects out to the embedding application
for error diagnosis and handling. There is a lot to be excited about in this.

Tonight I updated the embed_api branch to Parrot master. There were a few
conflicts and messes, but it was mostly painless. bluescreen is working to
implement some new features and start implementing some tests for the new API.
If things keep moving at this pace I think we are mergable within a week.
