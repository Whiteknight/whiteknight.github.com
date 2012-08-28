---
layout: post
categories: [Parrot, IO]
title: io_cleanup1 Lands!
---

FINALLY! The big day has come. I've just merged `whiteknight/io_cleanup1` to
master. Let us rejoice!

When [I started the project](/2012/05/27/io_cleanup_first_round.html), months
ago, I had intended to work on the branch for maybe a week or two at the most.
Get in, clean what I could, get out. Wash, rinse, repeat. That's exactly why I
named the branch "io_cleanup1", because I intended it to just be the first of
what would be a large series of small branches. Unfortunately as I started
cleaning I was lead to other things that needed to go. And those things lead
elsewhere. Before I new it I had deleted just about all the code in all the
files in `src/io/*` and started rewriting from the ground up.

Sometimes sticking with a plan and breaking up projects into small milestones
is a good thing. Othertimes when you know what the final goal is and you're
willing to put in the effort, it's good to just go there directly. That's
what I ended up doing.

To give you an idea of what my schedule was originally, I had intended to get
this first branch wrapped up and merged before GSOC started, so that I could
keep my promise of implementing 6model concurrently with that program. With
GSOC over last week (I'll write a post-mortem blog entry about it soon), I've
clearly failed at that. I'm extremely happy with the results so far and given
the choice I would not go back and do things any differently. The IO system was
in terrible condition and it desperately needed
this overhaul. I wish it hadn't taken me so long, but with a system that's so
central and important, it was worthwhile taking the extra time to make sure
things were correct.

Where to go from here? My TODO list for the near future is very short:

1. Threads
2. 6model
3. More IO work

The Threads branch, the magnum opus of Parrot hacker **nine** is 99.9% of the
way there. If we can just push it up over the cliff, we should be able to
merge soon and open up a whole new world of functionality and cool features
for Parrot. I'm already planning out all the cool additions to Rosella I'm
going to make once threads are merged: Parallel test harness. Asynchronous
network requests, an IRC client library. The addition of a real, sane
threading system opens up so many avenues to us that really haven't been
available before. Sure there are going to be plenty of hiccups and speedbumps
to deal with as we really get down and start to use this system for real
things, but the merge of the threads branch represents a huge step forward and
a great foundation to build upon.

I'm going to be putting forward as much effort as I can to getting this branch
wrapped up and merged. Some of the remaining problems only manifest on
hard-to-test platforms, which is where things start to get tricky. As I
mentioned in an email to parrot-dev a while ago, test reports on rare
platforms are great, but if we can't take action on the reported failures
we can get ourselves into something of a bind. The capability to find problems
on those platforms and the capability to fix problems on those platforms are
two very different capabilities. But, most of the time that's a small issue
and we're going to just have to find a way to muscle through and get this
branch merged one way or the other. If we can merge it without purposefully
excluding any platforms, that would be great.

Before anybody thinks that I'm done with IO and that system is now complete,
think again. There is still plenty of work to be done on the IO subsystem,
and all sorts of cool new features that become possible with the new
architecture and unified type semantics. I want to separate out Pipe logic
from FileHandle into a new dedicated PMC type. Opening FileHandles in "p"
mode for pipes is clumsy at best, and I want a more sane system. And while I'm
at it, 2-way and 3-way pipes would make for a great feature addition (we
can't currently do these in any reliable way).

The one thing that has changed most dramatically in the new IO system is
buffers. The buffering subsystem has not only been rewritten but completely
redesigned. Instead of being type-specific they are now unified and type
independent. Buffers are their own struct with their own API. Instead of
having a single buffer that is used for both read and write, handles now have
separate read and write buffers that can be created and managed independently.
I want to create a new PMC type to wrap these buffers and give the necessary
management interface so they can be used effectively from the PIR level and
above.

Finally, the `whiteknight/io_cleanup1` branch tried to stay as backwards
compatible as possible, so many breaking changes I wanted to make had to wait
until later. In the future expect to see many smaller branches to remove old
broken features, old crufty interfaces, and old bad semantics. We'll make
these kinds of disruptive changes in much smaller batches, with more space
between them.
