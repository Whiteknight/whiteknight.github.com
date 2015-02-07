---
layout: post
categories: [Parrot, Threading]
title: Threading Timeline Changes
---

A few days ago I had a short talk about concurrency with plobsing. I was
hoping that my blog posts about the topic would start a larger conversation.
It seems that they have been successful in this regard. I said that I wasn't
going to publish any more blog posts until I had something to write, and now,
only a few days later, I have plenty to write about. plobsing seemed to like
the ideas I described in general, although he had a few points to make. I'll
describe some of his points below, and then post some of the revisions to
my timeline especially which I am making in response.

First, he didn't think that our current system was so irrepairable and
misfortunate as I suggest it is. There are some things, at least a handful
of abstractions and considerations which the current system makes and it would
be a waste to reproduce. Plus, even if we did reproduce some of the
abstractions, such as the abstraction layer over Win32 and posix pthreads,
we would end up with something nearly identical for our efforts. I'm against
this thought in so far as I am not willing to spend my personal efforts on
fixing the old system, but I'm not necessarily against other people doing that
themselves. So long as we end up with a nice system for threading and all the
warts disappear, I suppose won't care too much how we got to that point.
Since I hadn't planned on doing any threading work until the end of GSoC
anyway, it really doesn't matter whether I would be willing to do the work
or not. plobsing on the other hand is interested in starting work on threading
topics sooner than I am, so whatever he chooses to spend his efforts on is
fine by me. He certainly has the skills to go about the project either way, so
whatever he is more comfortable doing should be just fine.

Second, he suggested that the timeline I discussed needed to be rearranged.
He certainly doesn't want to wait until the end of GSoC to get started, as I
was planning to do. He also wants to push forward OS threads and related
issues first. Other refactors, such as the interp/thread refactors, the
various deprecations, and the green threads can happen later. Specifically,
what he wants to do right now is to make Parrot thread safe, so that libparrot
can be used in a threaded environment such as an Apache webserver. Once the
Parrot core is thread safe, we can then start implementing OS threads
internally, and only expose concurrency functionality to users much later when
we have our internal house in order. My idea was to start exposing a new
interface to the users first, then using that interface to add new
functionality later. Peter's idea is to start getting the internal
functionality working first and expose that to the user later when we have
enough parts available to expose properly. Either idea works fine, and his
idea has plenty of merit. If we expose an interface to the user too early,
and there are problems with it, it will take extra time and effort to reverse
course and change things.

I do have some worries, that if we get threading working internally to Parrot
before we do the necessary refactors on the interp structure, that we may
put ourselves into a situation where we will want to avoid the pain of that
refactor in the future. That is, if we have something that "just works", we
will be hesitant to "fix what isn't broken". Maybe that's paranoia on my part,
but I do want to make sure that rearranging the timeline doesn't lead us to
taking a lazy and easier, but less beneficial in the long run, route. I'm not
going to avoid changing the plan just because I worry we might all be lazy
and unmotivated by correctness, I'm just putting this forward as an area of
concern. Separating out the interp into "global" and "thread-local" portions
is a necessary step we need to take, even if we can (and maybe should) avoid
it for now. It's critical for a variety of stability and performance reasons
in the future, although in the short term it really could be one of those
details that becomes easy to overlook. I really don't want to overlook it.

The basic timeline for concurrency in Parrot now is this:

1. Make the core of Parrot safe for use in threaded environments.
2. Implement OS threading, at least for internal and testing use only.
3. Make Parrot safe to use with it's own threads, including building an
   extensive suite of tests to validate and exercise threading and thread
   safety.
4. Do everything else, including tasks, messages, proxies, and whatever else
   is either unimplemented or only partially implemented up till this point.
5. Expose the new functionality to the user.

Obviously this new timeline contains a lot of gloss and hand waving, but it
does raise an immediate concern: How do we make Parrot thread-safe internally?
If we start with the assumption that Parrot is going to disallow data sharing
and direct cross-thread data updates, much of the system can be ignored.
Where we have problems are the few bits of global data, global variables that
are used in a handful of places (including singleton PMCs) the GC/PMC
allocator.

We don't have many global variables in Parrot, but we need to eliminate them
completely. This should not be too hard a task, all things considered.

In the final system, the interp will exist on one thread and will not be
directly modifiable on other threads. In the current system we have one
complete interp per thread, and one thread does not (and should not) be
talking directly to the interp of a different thread. We can consider the
interpreter to be more or less "safe" for the purposes of this exercise.

The GC is the big pain point in this discussion. There are two good solutions
that we can pursue in the long-term: Either we can implement a concurrent
collector which runs on a separate thread and expects a multithreaded
environment, or we can implement several mini GC cores and run one on each
thread without interaction between them. I prefer the concurrent collector
for most applications, but that's hardly something we can do immediately. What
we can do in the short term is to implement a system of locks for the
allocator and GC to prevent data corruption there. Cleaning up globals and
surrounding the critical portions of the GC with locks would go a long way
towards making Parrot usable in a threaded environment.

I don't like locks. I don't want to use them internally and my idea for
threading explicitly mentioned that Parrot would contain few if any, and that
they would likely not be automatically exposed to the user. However, if we
use locks as only a temporary measure to help Parrot become thread safe while
the rest of the necessary infrastructure is built up, I guess I can't be
militantly against that. I will try to demand, in so far as my demands carry
any weight, that those locks not be made permanent additions to the system and
that they absolutely not be exposed directly to the user, but I cannot in
good conscience attempt to prevent their use entirely.

Again, I worry a little bit about the "it just works" dilemma. I would hate to
see us get trapped because we've got a working lock-based system that our
users are relying on, and that we don't have the fortitude or the motivation
to fix it later. My original timeline was based on the premise that we could
implement things piecewise and do things in such a way that we don't provide
temporary hacks along the way that we might become reliant on. Of course, the
downside to that kind of paranoid planning is that the usable bits of
functionality don't start appearing until much later in the process, and we
are left with a Parrot which is completely not thread-safe for quite a long
time. Again I will say that my timeline was just a proposal and a conversation
starter, and I'm not married to it. I also want to repeat just once more that
the final design of concurrency in Parrot does not contain, nor make much use
of, locks.

What we need initially is to implement some temporary locks, which I suggest
(and plobsing agrees) that we should use existing libraries and existing
platform tools and not try to brew for ourselves. In the short term, this
will add at least an optional dependency on some external libraries. If you
configure Parrot without these libraries, and therefore without thread safety,
you will continue to use single-threaded Parrot as is.

I also want to point out that even if we have OS threads much earlier in the
sequence, that doesn't mean that we will expose their use to the user. My
design doesn't really allow for executing code directly on a Thread, but only
as part of a Task. It's likely that Parrot will have some internal-only
thread implementations for a while before the functionality is fully exposed
to the user.

Once we have locks in place inside the GC and allocator, fix up some of the
basic thread primitives and abstractions, and clean up the use of global
variables, we should be in a pretty good position to move forward to the next
steps: Implementing OS threads, implementing a new system of message passing
and thread-safe mailboxes (likely using an existing library), the necessary
interp refactors, and implementing a thread-safe or a properly concurrent
GC core. All those things in place, we can move on to the next steps:
read-only proxies, tasks, and signals/callbacks. Finally, we can create and
expose an interface for users to employ all these things.

So the relative ordering of things in the timeline has already changed
dramatically, and plobsing seems pretty keen to get started with some parts
of the work sooner rather than later. Also, the general idea has already
received some preliminary support from dukeleto and cotto, who are already
talking about ways to integrate the new system with Lorito. I suspect that
we won't need to add anything special to the new M0 spec to support
concurrency besides the existing mechanisms for dealing with objects (methods,
attributes, etc) and FFI. If we have those things, and that all-important
prohibition on direct data sharing, I think we should be good to go as far as
Lorito is concerned.
