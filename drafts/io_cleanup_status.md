---
layout: post
categories: [Parrot, IO]
title: IO Cleanups Status
---

Tonight I've hit something of a milestone with my branch to rewrite the IO
subsystem. As of tonight the parrot binary, `parrot-nqp` and `winxed` all
build in my branch and coretest runs (though fails some tests). The entire
build does not complete because of some failures related to dynops, but it
does get most of the way through. This means that most of the main-path IO
APIs and FileHandle operations are working correctly, which is a relatively
small portion of everything that has changed.

With Parrot building, I'm now able to more closely keep track of progress
and regressions, and do more live testing as I make new changes. Until this
point all my changes have just been mental exercises, so I'm happy to have
a little bit more feedback and even some validation.

Of course, just saying that it builds doesn't really mean anything. Several
things are still not implemented or completely wired up. Some operations on
files such as `seek`, `peek` and `tell` are still not implemented yet. Several
methods on the various PMCs (`FileHandle`, `Socket` and `StringHandle`) have
not been updated to use the new system. There are a few regressions I need to
address with regards to buffering. Specifically, "line buffering" has been
removed from the system during the rewrite and hasn't been added back. Line
buffering in Parrot has never really done much, but it's just hacky and
obscure enough that I'm sure somebody is relying on it.

Some things, like files opened for dual read/write modes or append modes
haven't been completely dealt with in code either. I don't think there's a lot
of work to do for this, but since the buffering architecture has changed so
much from what it used to be and since these modes are relatively rare and not
as thoroughly tested I want to spend a little bit of extra time making sure
there are no regressions.

Also there are several coding standards tests (especially for function-level
annotations and documentation) which fail spectacularly in the branch, and
it's going to take time to update all the old documentation and add docs for
all the new functions. I also need to update PDD 22 to reflect the new
architecture of the IO system.

I've been working on this branch pretty aggressively for the last two weeks
and I think I'm about 50% of the way done. That's not too bad considering the
magnitude of the change and the amount of time I've had to hack. Within a week
or two more, if all goes well, I think the branch might be ready for wider
testing and eventually merging.

As usual when we're talking about changes this big, merges are not something
to be rushed. Assuming all goes well and other people like what I've been
doing, expect to see a brand new IO system in Parrot sometime later this
summer.

