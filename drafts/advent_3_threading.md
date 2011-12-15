---
layout: post
categories: [Parrot, Advent2011]
title: Advent 4 - Threading
---

In GSOC two years ago my student Chandon started working on a very cool new
project for Parrot: Hybrid Threads. I hadn't really put too much thought into
the future of Parrot's concurrency architecture until that point and the idea
certainly seemed novel and interesting to me. We went ahead with the project not
quite sure how far he would get but very eager to see something happen with our
ailing threading system.

The problem, needless to say, ended up being much larger than a project for a
single summer and by the end of it Chandon had most of a green threads
implementation set up, but nothing that was quite mergable to master. His
branch sat, un-merged and un-molested, for a year before anybody decided to
take a second stab at it.

Several months after Chandon's work ended I sat down and started seriously
thinking about what I wanted Parrot's concurrency system to look like in the
future. If we could have anything we wanted, what is it exactly that we would
want? I started by reading up on all sorts of other technologies: Erlang,
node.js, stackless threads in python, and others. I even read a few tangentially
related materials, like some of the
[problems with common string optimizations][string_problems]. Ideas in hand, I
began writing a short [series of blog posts][threading_posts] describing my
personal conclusions. My plan for concurrency was very close to Chandon's hybrid
threads idea but with a few extra details filled in. I suggested that we should
stick with the hybrid threads approach, explicitly restrict cross-thread data
contention by using messages and read-only proxies, avoid locking as much as
possible, and rely on the immutability of certain data to keep things as simple
as possible.

[string_problems]: http://www.gotw.ca/publications/optimizations.htm
[threading_posts]: /2011/04/23/vision_parrot_concurrency.html

I didn't have the time to work on all this myself, at least not until several
other projects cleared my TO-DO list. This is where hacker **nine** comes in.
Nine wanted to work on adding threading to Parrot and liked the hybrid threads
approach and some of the ideas I had been working on. So he checked out a copy
of Chandon's green_threads work, updated to master, and started fixing things.
A few weeks later green_threads was merged to master with Linux support only.
I said I would work to port the system to Windows as well and nine would
push forward with the hybrid portion of the hybrid threads design.

We're not ready with that yet, but we are getting painfully close to a working
system. I've been stymied by the house hunt and the fact that I don't have a
windows system at home to play with. Nine has had a few infrastructural
difficulties, especially with GC, but he's making great progress regardless.

Here's an overview of the threading system we're working towards in brief: We
will have two layers of concurrency: The first are the Tasks, or "green
threads". Each Task represents an individual unit of executing work and multiple
Tasks can execute together on a single thread. They do this through preemptive
multitasking, the Parrot scheduler occasionally fires alarms and switches Tasks
on the current thread if there is more than one in the queue. Since Parrot uses
Continuation Passing Style internally, this mechanism is relatively simple to
implement (It's the alarms that are surprisingly difficult and not
cross-platform, but the actual Task switching is quite simple after that).

Multiple Tasks running on a single thread gives more of an illusion of
concurrency than the real thing, because you aren't making use of multiple cores
in your processor hardware or exploiting any context-switching optimizations
at the lowest levels of threads. What we will have to take things to the next
level is the OS threads implementation. Internally Parrot will maintain a pool
of worker threads. When you create a Task you will have an option about where
to dispatch it: On the current thread (useful where we need safe read/write
access to PMCs on the same thread without locking), on a specific target thread,
or on a completely new thread. Or, if you don't want to specify you can let the
scheduler dispatch the Task to the best thread.

When you think about what this kind of system enables, it's actually pretty
impressive: easy asynchronous IO (schedule an IO request Task on a dedicated IO
thread), easy event handling (schedule a new Task on the current thread to keep
data locality), easy threading (schedule a new Task, the scheduler will set up
the Thread for you), easy eventing loops (main thread reads event sources and
schedules Tasks), easy library callbacks (library callback schedules a Task in
the owner thread), auto-threaded array operations (schedule tasks for subsets of
the data, the scheduler puts the tasks on the threads with lowest latency) and
a variety of other modern techniques. This system, once it's completed and all
the bugs are ironed out, will really be a big boost for Parrot in terms of
feature set and usability.

I don't know when we will be ready to merge, but I can't imagine it will
happen before the 4.0 release. Sometime in early 2012 expect to see the new
hybrid threading implementation for Parrot, even if it only works on Linux
initially. By the 4.6 release I expect we will have a pretty robust hybrid
threading system available on all our target platforms, and several of our
HLLs and libraries will be making use of it.
