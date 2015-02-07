---
layout: post
categories: [Parrot, Threading]
title: My Vision for Parrot Concurrency
---

Threading is one of those things that is almost always on my mind, but it
seems like we never get anywhere with it. We have tons of questions about how
to implement threading, what particular mechanisms we want to use, and how
we want to share data and all sorts of other things. From those questions
we haven't taken the time to sit down, meld minds, and actually hammer out
answers. Without answers, we don't derive any particular vision of what the
systems will look like, and without knowing what the system will look like we
can't break that down into a roadmap so development work can begin in earnest.

Today, I want to start changing that situation. Today I'm going to start
laying out a vision that I have in my head for how concurrency could work in
Parrot. I'm putting some of these ideas out there not because I want this to
be accepted as *the* idea, or be the first idea and gain "mindshare" or
anything like that. I want to start the dialog, seed it with some of the
diffuse ideas we've been kicking around, and see if we can start coming
together around certain bits of it. Today I'm going to be giving a high-level
overview of the idea I have in mind, and will be willing to fill in some of
the many omitted details on demand, either in subsequent posts or in person.
I will also try to give a high-level overview on the steps necessary to
implement this new system starting from where we are now in Parrot 3.3.

Threading is not a super-huge priority for us, at least not compared to some
of the other things we would like to deliver between now and 4.0. I do think
we would be remiss, however, if we did not at least have concrete plans in
place by then. If we could have some real code written by then, all the
better. Maybe in a near-future post I'll talk about what some of our big
priorities as an organization are in the coming months.

Concurrency is a multi-faceted problem, and one that is solved differently by
different technologies. Because Parrot aims to support multiple different and
divergent languages and runtimes, the discussion tends to settle around two
possible polarized alternatives: Either Parrot provides very little in the way
of concurrency besides a bare handful of the most basic primitives, or Parrot
provides a system so flexible and grandiose that it is a complete superset of
any system which ever has or ever will exist. I suspect that we don't need to
go to such extremes, but instead we can try to find a middle ground. We want a
system which, by default, is useful and flexible. We also want to leave some
of the particular details up to the HLL or runtime developers. I don't think
that Parrot needs to necessarily provide any kind of guarantee of memory
consistency in all cases, for instance, and we also don't need to provide
huge libraries of things like locks, mutexes, semaphores, etc. What we should
do is provide a system which is usable and performant by default, but leave
open opportunities for HLL and library developers to build their own tools
on top of it. We can accomplish this through delegation, use of subclasses,
and HLL mapping, among other mechanisms.

## Hybrid Threads

Last year for GSoC, student Chandon proposed and began implementing a very
ambitious plan for a hybridized threading approach. At the time I viewed the
whole idea as an interesting curiosity, but the longer time goes on the more
I feel like the best answer for us does lay somewhere down that path. I've
introduced and discussed on this blog several of the important concepts
involved in this idea, and I won't repeat myself in this post. Going forward,
I'm going to assume the reader knows all about threads, the difference between
"heavy" OS threads and green threads, the ideas behind concurrency primitives,
the problems with deadlock, resource contention, data inconsistency, and
anything else I feel like talking about.

First things first, my idea involves a hybridized threading approach similar
to what Chandon proposed for his GSoC project. The hybridized thread approach,
which I will refer to as "N:M" for now (N OS threads serving as the substrate
for M continuation-based green threads) I think will provide us with the most
flexibility. In the simple case where `N == M == 1`, we have a single-threaded
system like what Parrot is right now. In the higher case where `N == M > 1`,
we have something like posix threads where each "Thread" in Parrot corresponds
to a unique thread at the OS level. In the decoupled case where `N != M`, we
have a system similar to Erlang's actors or Python's Greenlets. I think that
this basic system provides us plenty of base usability and flexibility without
having to either provide too much, or too little.

From this point forward I will use the term "thread" to refer to the
underlying OS thread on which Parrot executes (the "N" in the calculus above),
and "task" to refer to the individual Parrot execution context ("M", above).

## Locks and Data Sharing

The system I envision is "locked" in the sense that an individual task lives
on a particular thread and cannot be moved to a different thread. While such a
movement mechanism could be possible if we were willing to put in the extra
work, I see no great overall benefit to providing it. If anything the
repurcussions of such a mechanism would be detrimental to the common case for
only small benefits in the rare case. Every thread contains a stable of tasks
which it owns and has exclusive rights to. Tasks in a single thread can freely
share data between each other without fear of internal corruption. In terms of
memory corruption or inconsistency I'm going to differentiate between
"internal" problems which manifest internally to the VM itself and would cause
system instability, and "external" problems which happen at the PBC level or
above and only cause algorithmic instability. For instance, data corruption
internally in the `PMC*` structure is far worse in many ways than
inconsistency between attributes in a PIR-defined object. In essence, I'm
differentiating between problems that could cause segfaults from the problems
that could cause exceptions. The former should be prevented by Parrot as well
as possible, the later should be prevented by the user. If the user wants to
ensure that several operations occur together in a Task as an atomic block,
she can tell the parent Thread to disallow Task switching during the "critical
section", perform all updates, then re-enable preemtive Task switching. Or,
she can implement a library of locks to prevent multiple tasks from running
similar operations at the same time, or...

The issue of data sharing between separate threads has been a problem which
has been dogging Parrot for as long as I can remember. In fact, this is one
question which was avoided completely not only by Chandon in his GSoC project,
but also by myself and by then-architect Allison and the rest of the community
as well. We all decided to punt the ball instead of trying to figure out how
Parrot should implement data sharing between threads, which goes to show just
how difficult of a problem it is. My vision for concurrency in Parrot does
provide a basic answer to this question, one which gives plenty of opportunity
for the user to get herself into trouble, but also provides the flexibility
and pluggability necessary to build more complex systems on top of it. My
answer, in short, is this: By default, Parrot shares no data between Threads.
We can share data between Tasks freely like I mention above, but we do not
share data between Threads.

Instead of sharing data directly between threads, we have a handful of
possible mechanisms. At the most basic level, we use a series of read-only
concurrency proxy objects which provide read-only access to data across
threads. As I'll describe below we will also provide a message-passing system
for "safe" data updates, and we will also provide direct, unprotected, access
to data if the user specifically requests it. We do run into the possibility
that a read access may occur when the PMC is in an inconsistent state with
respect to it's attributes. This is fine, and Parrot isn't going to bend over
backwards to prevent this. If the user wants to implement some kind of
mechanism to prevent inconsistent data reads, she can provide it herself. A
custom subclass of the concurrency proxy could provide a local cache of data,
or could detect inconsistent states in the target object and delay the read.

## Concurrency Proxies

Every concurrency proxy object contains a pointer back to the "original" copy,
and intercepts any attempt to modify data. If the proxy is locked, attempts to
modify data throw an angry exception. If the proxy is unlocked, write requests
will be turned into messages and sent to the mailbox of the owner thread. Of
course, the user can subclass or HLL map the concurrency proxy object to
provide custom behavior instead, if needed. This idea of read-only
intercepting proxies is nothing new or difficult, Rosella already provides
these kinds of proxy objects, and implements them in only a few dozen lines
of Winxed code. It wouldn't be too much of a hassle to implement something
similar inside Parrot too. We could add four new ops to help manage data.
`lock` and `unlock` ops could be used to change the behavior of mutator
actions in the concurrency proxy object. A `lock`ed object disallows
modifications to the target object and throws exceptions where they are
attempted. An `unlock`ed one can update the target, either directly or by
sending an update message to the mailbox of the Thread owner of the target
object. Either behavior is acceptable to me as the default, and the other
should be easy to override in an HLL map if necessary. In this way, we could
even forego the two ops, and instead provide one as the default and allow the
other to be provided by an HLL. The second set of new ops would be `share` and
`unshare`. The first would take a PMC and return a concurrency proxy for it
(in the "locked" state by default), and the later would take a concurrency
proxy and return a direct reference to the target object from it. People who
want the built-in safety can use the proxies directly. People who don't want
built-in safety but do provide their own library of mutex and critical section
objects may want to live a little bit more dangerously and play with
cross-thread references directly. It's not our job to keep people from getting
themselves in trouble, so long as we provide decent defaults to fall back to.

In the case of an unlocked proxy, I don't care whether data update messages
are passed synchronously or asynchronously. Either we can pick a default and
allow the user to override in an HLL mapped subclass, or we can provide a flag
or a callback as a second argument to `unlock`. Synchronous update operations
are probably the most familiar, but asynchronous ones are more flexible and
would have better throughput. Plus, we can define the synchronous ones in
terms of asynchronous operations, especially if our system of scheduling and
managing Tasks is flexible enough. In the asynchronous case we send off the
message and terminate the current Task. Then we schedule a callback Task to
pick up when the operation succeeds. I won't get into too many details here,
but we do have plenty of options.

## Tasks, Threads and Interpreters

Every Task is a tuple of a Context object, a Continuation, a Mailbox and maybe
an array of startup invariants, similar to how the main program args are
currently stored inside the interpreter object. Actually, we may have a
Mailbox at the level of the Thread instead of at the level of the Task. That
might be better. Every Thread object contains an array of Tasks, a pointer to
the current Task, and scheduler logic to control when and how the next Task is
launched. A handful of simple operations would control when and how Tasks were
switched. In the simple `N == M` posix-ish case, the thread disallows Task
switching. In a case with cooperative multithreading, we disallow automatic
Task switching but do allow the user to manually specify a Task switch. A
cooperative approach where Tasks must be manually scheduled using
`thread.yeild()` or `thread.execute(task)` calls is trivial to do. New tasks
can be added using `thread.schedule(task, priority, args, ...)`. These names
are just speculative of course, I'm just trying to show that tasks are managed
either through or with particular threads. Somewhere along the line I suspect
we are going to have a ThreadScheduler which keeps track of all running
threads in the interpreter, can create or remove threads, can get the current
thread, and do a handful of other tasks as well. Threads belong to the
ThreadScheduler, Tasks belong to their respective Threads. The system in this
regard is open but rigid.

That brings me to the idea of the interpreter. There will still be one, though
its capacity is going to be diminished. Much of what currently exists in the
interpreter will be refactored out into the Contexts. This is part of the
Lorito plan anyway, so there should be no surprises here. The interpreter will
handle some global information such as the global namespace store, the global
class store, and maybe the GC. I don't care at this point how we choose to
implement the GC in this situation. Either we could have a single monolithic
GC and every time we want to run mark & sweep we need to freeze all threads,
or we could have a single GC where each thread gets its own set of pools. Or
we could have multiple GC cores because overhead there isn't so big a problem.
I suppose there are a lot of tricky technical details in the implementation,
but right now I don't suspect any particular approach would be worth
considering in earnest. If the GC runs in its own thread, and is the only
thread which is allowed to read and modify certain flags in the PMC structure,
it shouldn't be too hard to allow a concurrent collection to take place
without having to block all other threads in the system. Notice also that with
a tricolor system we could block one thread at a time to perform all necessary
marks in sequence, then sweep at the end when all the threads are running
again. We have plenty of options here and I'm not married to any particular
idea just yet. GC is one of those details that we can worry about after we've
made some of the other tough decisions and settled on some of the bigger
ideas.

The interpreter will be a global object which contains master copies of
certain bits of data. Each Thread will itself be like a mini-interpreter,
containing local caches of proxies to that data. The whole idea centers on
this use of cheap, read-only proxies to handle data sharing between threads.
The benefit of these, since they intercept vtable calls, is that they can be
used to lazily create new read-only proxies for nested data in complex
structures. For instance when we create a new thread we do not need to clone
the entire tree of NameSpace objects. Instead, we can create a single
concurrency proxy for the root of the tree, and allow it to lazily create
proxies for subnodes as they are searched. And because Tasks in a single
Thread can share data without penalty, we would only need a single cache of
such "global" data proxies per Thread, not per Task. When we look up something
global, such as a Class for instance, we ask the current Thread object, who
either has the "live" version of the data (in the case of the "master" Thread)
or it asks the local proxy which retrieves it from the live version directly.

## Messages

Through a proxy you cannot update data on a different thread directly, locked
or unlocked. Instead, we would have to pass messages. At the moment I conceive
of messages as being small, internal-only commands to update particular values
or particular sets of values across threads as an atomic unit. I suppose
there's nothing stopping a message-passing system from being expose to the
user and used in a variety of ways. In that case, there would be small
internal-only messages which did simple data updates, and larger command
messages which could be user-defined to trigger actions on different threads.
In either case, assuming we make the guarantee that Tasks on the given Thread
do not preempt during parsing of messages, we can end up with a pretty simple
and reliable system. In all cases the interpreter "lives" on the master
thread, which is the first thread created. Accesses to the interpreter and its
store of global data from the master thread is direct and fast. Because of
this inequity, best practices suggest that the master thread should be
reserved in a multi-threaded system for operations which affect the global
data store. Other operations should happen somewhere else less contentious.
Again, this is a best practice and a cultural decision, the user can perform
any actions they want on the master thread and bog the whole system down
with locks and crap if they want to do it that way.

At regular intervals, either preemptive or cooperative, a thread can check
its messages and invoke them if there are any in the mailbox. This is how
simple data updates happen across threads in the common case.

A Message can really be any number of things. In the most simple and flexible
case a Message would be a combination of an invokable and data. In the case of
a simple attribute update, the invokable part of the data would look like this
(in Winxed):

    function update_property(var target, string attribute, var value)
    {
        ${ setattribute target, attribute, value };
    }

A scant handful of similar stock message types, along with the ability to
define custom ones provides all the flexibility we would need. The user would
pass a message by calling a method on the Thread object:

    thread.send_message(func, ...);

On the receiver side, the Thread would check its messages between Task
switches. We check the messages, and if there are any in the mailbox we
execute them first before we activate the next task. If we have no pending
tasks, we check the mailbox in a short loop or on a delay timer or something.
I'm not certain that Messages and Tasks need to be different. In fact, I'm
certain that they can be the same kind of object. The only difference, from
the point of view of the Thread, is where the Task is stored. In the case of
a Message, it is stored in the Mailbox. Once the Message begins executing, it
is treated as a normal task and can be preempted to run other Tasks. In fact,
this is the mechanism by which Threads can be populated with Tasks in the
first place, and provides a very natural way to farm tasks out to worker
threads.

## Review

Here's a short recap of some of my ideas:

1. We use an N:M hybrid threading system, where greenlet-like Tasks are owned
   by posix-ish Threads.
2. Tasks are tied to Threads and cannot be moved, although new tasks can be
   created, scheduled, or deleted at any time.
3. By default, data can be shared seamlessly between Tasks in a single Thread
   without worry of internal corruption. Mechanisms could be added by HLLs and
   libraries to prevent this, however.
4. By default, data is shared read-only between Threads. A concurrency proxy
   object restricts write accesses, and wraps read accesses lazily in new
   proxies.
5. concurrency proxies can be locked and unlocked. A locked proxy prevents all
   data updates across threads. An unlocked one allows updates to be performed
   via message passing (possibly synchronous or asynchronous).
6. If the user wants, she can get direct access to data across threads, at her
   own risk.

And here are some of the things that I particularly like about this system:
1. We can easily collapse down to 1:1 posix-alike threads. An external library
   or HLL could easily subclass the ThreadScheduler or set up some other kind
   of wrapper interface to provide posix-like threading, and provide a library
   of familiar routines for managing them.
2. The system works without locks, because we don't allow cross-thread data
   contention by default. Of course, the user is free to disregard our
   protection mechanisms and provide a library of lock primitives of their
   own.
3. Using read-only proxies as our data sharing mechanism saves us from having
   to go overkill with a system like STM, which isn't needed for the common
   case. Tasks on a single thread have no data sharing overhead, and updating
   data across threads exposes a message-passing interface which can be
   provided to the user for custom applications.
4. Parrot provides all the necessary underlying tools, and gives the user
   plenty of rope to trip over if necessary. We don't need to pessimize the
   common case by using all sorts of locks internally or using STM. The user
   has the ability to subclass, HLL Map, and use other techniques to implement
   most higher-level or alternate behaviors. Parrot provides good defaults,
   but lets the user customize those as much as necessary.
5. No matter what the user does at the HLL level, libraries and frameworks can
   always fall back to the old behavior: cross-thread data is read-only by
   default, messages can be passed to automatically update values without
   needing locks, etc.

These are the basics of my vision for concurrency on Parrot. Obviously this
plan is incomplete and not many of the details are filled in, especially not
at the fine-grained level. It's my sincere hope that we can start some serious
talks about this topic in the coming weeks and months, and we can be prepared
to start work on this system sometime within the next year.
