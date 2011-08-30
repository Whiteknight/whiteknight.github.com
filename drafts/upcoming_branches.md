We've got a couple big branches which could be merged before the next release
in the works, and I want to give a quick overview of each. If all of these
merge in before the 3.9 release, it could end up being a pretty awesome
release overall.

## `whiteknight/frontend_parrot2`

The `whiteknight/frontend_parrot2` branch is one I started to look into the
reoccuring idea I and others have had for some time: Can we bootstrap into PIR
earlier and rewrite the frontend executable to be at least partially written
in PIR?

The answer is a resounding "yes!". The only things that are done in C are
creating the interpreter, creating the IMCC compiler object instances, and
parsing a handful of the lowest-level commandline options. Everything else
is done by a new PIR program called 'prt0.pir'. The prt0 program parses the
rest of the commandline options and is in charge of doing things like reading
in and compiling PIR, loading PBC packfiles, writing out PBC files, handling
uncaught exceptions, showing usage, version and help text, and doing a few
other things. In the future when IMCC is dynamically loadable, the prt0
frontend will also be responsbile for loading it and registering the compiler
objects.

What does it mean for you? Well, startup performance is affected in a small
way. For simple programs, startup is slightly slower. For more complicated
programs, startup is slightly faster. It gets faster as the program gets
more complicated, but we're only talking about startup times. Once we start up
and get to running the main program logic, there should be no change in
performance.

In the bigger sense, we're running PIR code *before* we execute the `:main`
subroutine. This means that we can start defining things in PIR, like classes
or global functions, or whatever. It means we also have complete control over
things like packfiles from PIR, so we can start changing around some of the
IMCC internals to make better assumptions about where it's being called from,
etc. That should help to simplify some code and provide us with new features.

Parrot hacker plobsing has been doing some excellent fixing and cleanup work
in this branch, and barring a few last minute problems we should be good to
merge it sometime this week. Most users won't even notice the change, but it
will enable us to start working on other cool new projects.

## `whiteknight/kill_threads`

We've finally hit our patience limit as a community, and we're ripping out
threads. At least, we're doing it in a branch. It's not all parts of the work
are straight-forward, but already I've been able to clean up a lot of terribly
ugly code and accidentally fix a few bugs along the way.

We've been living with a lot of problems in the threading system, and alot of
problems tangentially caused by its implementation for a long time now.
Ripping it all out is like a breath a fresh air. You almost start to get
hopeful and think "hey, we could really have new concurrency stuff implemented
here, if we don't have to work around all this other junk".  We've got a lot
of ideas floating around for a replacement, and a few hackers who have
expressed an interest in implementing a replacement. I'm hoping that as soon
as we get the crufty old code out of the way we will be able to break ground
on the shiney new stuff.

This branch has a ways to go. There are still a few broken tests, including
a few which aren't entirely straight-forward to fix. Also, I'm, not entirely
certain where we stand with respect to deprecation boundaries, so we might
not be able to merge it until after 3.9 in any case.

## `whiteknight/6model`

I started this branch last weekend, to start porting the 6model infrastructure
from the NQP repo to Parrot. Right now, it's basically a copy+paste of the
code to Parrot, plus fixes to make it build and start passing some of our
code formatting tests. This isn't necessarily the way we want to proceed in
the long run, we may want to do a proper port and rewrite the functionality
from the ground up with our own requirements in mind.

At the moment the branch builds but has a few problems with regards to
initialization. Also, I haven't done any testing to see if the 6model code
actually works. I suspect there will be some problems. We've ported over all
the PMC types and all the ops, so now it's a matter of seeing how things work
and fit together. I've heard jnthn, the author of 6model, is away for a few
days, so this branch might be on hold until we have a chance to talk to him.
