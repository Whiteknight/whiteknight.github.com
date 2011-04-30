---
layout: post
categories: [Parrot, Threading]
title: My Vision for Parrot Concurrency Redux
---

I want to revisit my
[post about my ideas for concurrency in Parrot][concurrency_post] and try to
add a bit more clarification about a few points.

[concurrency_post]: /2011/04/23/vision_parrot_concurrency.html

My whole vision for that system is based on a few basic precepts:

1. We do not want to provide the bare minimum, nor do we want to go crazy
   providing every feature ever conceived. We want to provide a sane default
   and a system on which other features can be built.
2. We want to provide a hybrid threading system so that approximations to
   "heavy" posix-like threads and "light" erlang-style actors can be built.
3. Various bits of the system should be encapsulated in objects, which the
   user should be able to customize, modify, adapt, or completely replace if
   possible.
4. We only go so far as to guarantee that the system will be *stable* in the
   sense that we avoid internal data corruption, crashes, and segfaults. We
   do not guarantee any sort of timely consistency at the level of objects.
   The user is responsible for these things. Your program might throw weird
   exceptions, but it won't segfault.
5. We will guarantee *eventual consistency*, in the sense that update commands
   and cross-thread messages will be delivered and acted upon, although not
   under any particular time constraints, by default.
6. We do not allow direct cross-thread data modifications by default, but we
   also do not prohibit them if you absolutely want to do something bad.
   There will be an option to enable you to really hurt yourself, if you want.
7. Data access and modifications between threads are never immediate, and we
   won't pretend differently.
8. Have a system able to scale well to multi-core processors and parallel
   hardware, but also be able to degrade gracefully in non-parallel
   environments.
9. Avoid the need to use locks internally. Absolutely, positively avoid the
   need for a global interpreter lock.

Together these things start to inform a concurrency system design which is
flexible and powerful but also light-weight by default. Let me go over some of
these items in more detail.

For the first point, we want to avoid extremes. Some papers and systems
advocate that we create a minimalist system and only provide a scant handful
of "concurrency primitives" upon which everything else can be built. This is
a fine approach, and easy enough to implement, but doesn't do anything to
provide a default solution to projects just trying to get started without
needing to implement threading themselves. It also doesn't help us decide what
default we should build on top of those primitives. The big point of Parrot is
to provide this common platform for compiler and project developers to use to
get a jump start. We implement things so you don't have to. We would rather
provide a basic system which users can customize to fit specific semantics
rather than provide only the tools for building a system and expect everybody
to build their own from the ground up.

Big, well-staffed projects should have the ability to throw the defaults out
and roll their own to meet their own particular needs. Rakudo does this with
everything from the object model to the subroutine dispatcher. It's not
anything special for us to make the system pluggable. If anything, it's
expected. Smaller projects who don't have enough manpower to throw the baby
out with the bathwater and roll their own concurrency system should be able to
use what we provide to get moving immediately, even if what we provide isn't
100% perfect in the long run. I suspect it will be good enough, with some
tweaking and configurability, for most uses.

The second point is a continuation on the first idea. We want a system which
can appear to be heavy-weight like posix, or more light weight like what
Erlang provides with its [Actors][] model. By providing a flexible hybrid
system, we can allow users to approximate both these systems and everything in
between. They can tweak the parameters to configure it as necessary. We could
revamp the current threading system and only employ heavy-weight threads, and
allow users to cobble together a million implementations of tasks on top of
that, but we lose a lot of flexibility at the VM layer and we force a lot of
people to implement for themselves something that we could easily provide by
default. And I think we do want to provide this, because I think people
definitely want it. Have you ever heard of Python's [stackless][] threads, or
[greenlets][]? PHP doesn't have many [threading][php] options, but don't you
think that a PHP compiler that offered lightweight task multiplexing on an
externally-limited thread pool might make great sense? My point is that there
are both existing users of light-weight tasks, and many potential users
of them in the future. We could go the easy route and only provide heavy
threads, but I feel like that isn't an acceptable default for what we want
to accomplish in the future.

[Actors]: http://en.wikipedia.org/wiki/Actor_model
[stackless]: http://www.stackless.com/
[greenlets]: http://packages.python.org/greenlet/
[php]: http://blog.motane.lu/2009/01/02/multithreading-in-php/

The third point follows in a similar vein. We want the system to be
configurable by the user. The best way to allow that in current Parrot is to
wrap up behaviors in objects and allow the user to subclass, delegate, or
HLL map new types to provide necessary behaviors. We can allow all these
mechanisms, or pick and choose which works best on a case-by-case basis. I
don't have any strong opinions on the matter, and I suspect most of the
details will become apparent as we get further into the design and
implementation of this system (if this is indeed the path we want to travel).
Some things, such as a global ThreadScheduler type wouldn't be amenable to
HLL mapping, but certain aspects of its algorithm could be open to
modification by setting parameters or delegating behaviors to subtypes which
the user has control over.

The next couple points all deal with the elephant in the room: data
consistency. I am expressly denying the need for immediate global consistency
between data visible on all threads. That is not the way the world works and
other VMs have gotten into huge trouble trying to pretend otherwise. We have
two options when trying to write a single chunk of data from between two
threads: Either we can demand that the updates be globally visible as
immediately as possible, or we demand that the data be consistent and safe at
all times. This second option is the one that I favor. If we provide a basis
for a concurrency system which we can make certain consistency guarantees
about, we can use that to build all sorts of things on top of. We should
not attempt to guarantee immediate data consistency. In Java, if you write
data in one thread on one clock tick, there is a guarantee made that any other
threads which read that data in the exact next clock tick (as measured in
nanoseconds, not the actual processor clock) will see the update. This
guarantee leads to, among other weirdness, the need to change the logic of the
clock to give enough time for such "magical" writes to actually happen. In
Parrot we will be a little bit more realistic. Data isn't even guaranteed to
propagate from processor cache to RAM in a single clock tick. If you want a
guarantee, provide it yourself. You can put your threads into a wait loop
until the update has been processed and a receipt has been received, or you
can enter a polling loop, or you can use a system of locks (locks which you
will probably be writing yourself), or you can use a system of callbacks, or
whatever. If you want immediate data propagation you are out of luck and
there is nothing I can do to help you with that.

A complaint that I can foresee receiving goes something like this: "The
JVM/.NET/whatever VM allows me to write data between threads immediately, if
Parrot doesn't do that, it's a design flaw and Parrot is inherently inferior."
My reply is that if we wanted to do the same things as the JVM or .NET, just
as well as they do them, we would be writing software for the JVM or .NET
instead. They are already very good at what they do and how they do it, and
trying to emulate their behavior is not a design goal for Parrot. Nor is it a
winning strategy for us. This threading system that I am proposing for Parrot
will be different from offerings on other virtual machines. In some ways it
will be better. In others, worse. To get the most out of it, you are going to
have to use different techniques and algorithms than you would use on a
different platform. You're going to have to think about things differenty. If
you want to solve a problem in the standard Java or C# way, maybe you need to
use Java or C# instead. They're fine languages, if you want to do what they
do in the way that they do them.

Data consistency is a similar problem. We can't guarantee that when you read
data from across threads that it will be consistent. The data could be in the
middle of an update, or there could be a series of unread update messages in
the queue, waiting to be processed to make modifications. We can guarantee
that data will eventually be consistent, in the sense that all sent messages
will eventually be processed. Again, if you want guarantees here you can make
them yourself. Actually, Parrot could add in some safeguards pretty easily
without hurting anything. It would be trivial to tell a thread to avoid
checking messages for a certain section of code, and then to implement the
threading system in such a way that task switching did not occur inside a
message. In that way, all messages are handled atomically, and if we bundle
our updates accordingly we can guarantee that they all happen together without
interference. Of course, that opens the door to "impolite" programs passing
very long-lived messages which starve out other tasks on the thread. Instead,
maybe we only say that internal-only update messages are not preemptable, but
user messages are. That works out much better, but we would need to add an
API to define a series of inter-related updates to be processed as a single
transaction bundle, and it requires us to have two message types instead of
just one. `transaction.begin()` and `transaction.commit()` and similar
routines. It's not unthinkable, but it is a little bit more "stuff" than I
would like to have as part of the base system. Keep in mind that I'm not going
for minimalism, but I'm not going to maximalism either.

Things like this really strike me as being rich fodder for future GSoC
projects. If we're far enough along next summer I'll definitely raise some
of these ideas again.

Reading data across threads is also problematic because reads could be
inconsistent. This is nothing that you can't work around, especially if you
are patient. Writing data between threads can be down-right dangerous because
you start to talk about data corruption. Because of this problem, and because
we don't want to be providing heavy-weight solutions like STM by default, we
disallow write access to data which lives on a different thread. Instead, as
I've mentioned before, we use message passing to tell the target thread to
update the target data at the next available convenient time. If you
absolutely want a direct object reference and you demand to be able to do
direct cross-thread data writes, I won't be held responsible for what happens.
You'll come to our chatroom to say "HALP! I CAN HAZ T3H S3GFAULTZ!!", and I'll
reply that "UR doin it wrong".

Other VMs like Java or .NET don't offer a guarantee against data corruption.
Instead they provide an elaborate system of locks and caveats. If you do
everything perfectly by the book, your data probably won't get corrupted.
You'll lose a few percentage points off the top of your performance metrics,
but you won't corrupt any data. More likely, if you do something imperfectly,
data will get corrupted and you will run into extremely strange runtime
errors, program crashes, and other problems. I've spent more hours of my life
debugging these kinds of things, and if I could have that time back I would
take it in a heartbeat.

What I'm suggesting here is that we trade data immediacy for data consistency.
It's a system I think we can use to great effect, especially since we've seen
the Erlang people and the functional programming people use these kinds of
ideas for a long time to great effect.

Parrot, on the other hand, will allow you to send update messages and avoid
locks at the expense of possibly having a longer delay between the time you
send the update message and the time the data is actually updated. The chance
of running into catastrophic failure are lower because we take the triggers
for such failures off the table. We provide several other avenues for getting
yourself into trouble, but we severely limit your ability to corrupt data
through cross-thread writes. Everything is a tradeoff, and I feel like this is
a good decision for a number of reasons.

Point #8 is an important one. We need Parrot to be able to scale, because we
don't know what kinds of programs people are going to want to use it for, or
what kinds of hardware they will be running on, or even what programming
languages and threading semantics they will be using. We need Parrot to be
pretty flexible to cover most, if not all, use cases. The basic idea that I
am thinking about is this: Create tasks for the things you need to have done.
Then, farm those tasks out to execute on a variety of worker threads. You can
create as many or as few worker threads as you want and farm as many tasks
out to each as you want. Here's a quick code example to show what I mean:

    for (var data in all_data) {
        var task = TaskFactory.CreateProcessingTask(data);
        var thread = ThreadManager.GetBestAvailableThread();
        thread.schedule(task);
    }

Tasks are cheap to create because they are basically just an array and a
continuation. Once we create them, we schedule them on a thread to execute.
If we are on a huge high-performance cluster with thousands of processors,
maybe each task gets its own thread to run on. If we are on a small, single
processor system, maybe all tasks run on the same thread. In either case, the
loop above can stay exactly the same. What changes is the logic in
`ThreadManager` to choose what thread from the pool constitutes the "best"
one.

Think about a webserver, where we are getting hundreds of hits per second.
Instead of needing to spawn a thread for each request, we have a fixed number
of threads and instead spawn a light-weight task for each request. It's a
pattern that is both not new, and also not known to have peformance problems
or limitations.

