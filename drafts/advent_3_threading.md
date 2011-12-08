---
layout: post
categories: [Parrot, Advent2011]
title: Advent 3 - Threading
---

In GSOC two years ago my student Chandon started working on a very cool new
project for Parrot: Hybrid Threads. I hadn't really put too much thought into
the future of Parrot's concurrency architecture until that point and the idea
certainly seemed novel and interesting.

The problem, needless to say, ended up being much larger than a project for a
single summer and by the end of the summer Chandon had most of a green threads
implementation set up, but nothing that was quite mergable to master. His
branch sat, un-merged and un-molested, for a year before anybody decided to
take a second stab at it.

After Chandon's work ended I sat down and started seriously thinking about
what I wanted Parrot's concurrency system to look like in the future. If we
could have anything we wanted, what is it exactly that we would want? I
started writing a short series of blog posts describing the conclusions I came
to, and started trying to drum up support from other community members. My
plan for concurrency was very close to Chandon's hybrid threads idea but with
a few extra details filled in. I suggested that we should stick with the
hybrid threads approach that Chandon was working on, explicitly restrict
cross-thread data contention by using messages and read-only proxies, avoid
locking as much as possible, and rely on the immutability of certain data
to keep things as simple as possible.

I didn't have the time to work on all this myself, at least not until several
other projects cleared my TO-DO list. This is where hacker nine comes in. Nine
wanted to work on adding threading to Parrot, and liked the hybrid threads
approach and some of the ideas I had been working on. So he checked out a copy
of Chandon's green_threads work, updated to master, and started fixing things.
A few weeks later, green_threads was merged to master with Linux support only.
I said I would work to port the system to Windows as well, and nine would
push forward with the hybrid portion of the hybrid threads design.

We're not ready with that yet, but we are getting painfully close to a working
system. I've been stymied by the house hunt and the fact that I don't have a
windows system at home to play with. Nine has had a few infrastructural
difficulties, especially with GC, but he's making great progress.

I don't know when we will be ready to merge, but I can't imagine it will
happen before the 4.0 release. Sometime in 2012 expect to see a new hybrid
threading implementation for Parrot, even if it only works on Linux initially.
