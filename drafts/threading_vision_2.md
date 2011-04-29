I want to revisit my post about my ideas for concurrency in Parrot and try to
add a bit more clarification about a few points.

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

Together, these things start to inform a concurrency system design which is
flexible, powerful, but also light-weight by default. Let me go over some of
these items in more detail.

For the first point, we want to avoid extremes. Some papers and systems
advocate that we create a minimalist system and only provide a scant handful
of "concurrency primitives" upon which everything else can be built. This is
a fine approach, and easy enough to implement, but doesn't do anything to
provide a default solution to projects just trying to get started without
needing to implement threading themselves. The big point of Parrot is to
provide this common platform for compiler and project developers to use to
get a jump start. We implement things so you don't have to. We would rather
provide a basic system which users can customize to fit specific semantics
rather than provide only the tools for building a system and expect everybody
to build their own from the ground up.

The second point is a continuation on the first idea. We want a system which
can appear to be heavy-weight like posix, or more light weight like what
Erlang provides with its Actors model. By providing a flexible hybrid system,
we can allow users to approximate both these systems and everything in
between. They can tweak the parameters to configure it as necessary.

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
between data visible on all threads. That is not the way the world works, and
other VMs have gotten into huge trouble trying to pretend otherwise. We have
two options when trying to write a single bit of data from between two
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
just one. It's not unthinkable, but it is a little bit more "stuff" than I
would like to have as part of the base system. Keep in mind that I'm not going
for minimalism, but I'm not going to maximalism either.

Reading data across threads is problematic because reads could be
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

Parrot, on the other hand, will allow you to send update messages and avoid
locks at the expense of possibly having a longer delay between the time you
send the update message and the time the data is actually updated. The chance
of running into catastrophic failure are lower because we take the triggers
for such failures off the table. We provide several other avenues for getting
yourself into trouble, but we severely limit your ability to corrupt data
through cross-thread writes. Everything is a tradeoff, and I feel like this is
a good decision for a number of reasons.

Precept #5, for
instance, means that we do not need to provide any particular mechanism for
guaranteeing data consistency between threads, such as STM. Users can build
STM on top of Parrot, or we can provide STM and have it available as an
option, but we don't need to have it and use it by default. We also don't need
to provide a selection of locks, mutexes, semaphores, critical sections,
spinlocks, or other primitives which other threading systems may provide.
Again, these things can be built on top of Parrot or offered as an option, but
they don't need to be part of the base system.

By making accesses to cross-thread data read-only by default, we can have
huge complexity and overhead savings. Consider the case of the global
NameSpace heirarchy. NameSpace objects form a tree, the root of which is
located in the interpreter. This tree contains not only the NameSpace objects
themselves, but also the global data items stored directly in the NameSpaces
such as non-method Subs. In the current threading system when we create a new
thread we must clone the interpreter, which means we must deep-clone this
entire tree. That's a huge waste of both performance and memory. Plus, it
creates a huge synchronization problem. If we have two copies of the NameSpace
tree and each thread starts making modifications to it, we can end up with
programs which are extremely strange to say the least. The contents of a
NameSpace change depending on which thread you are executing in. This is all
not to mention that other global items like Class definitions would be unique
to the thread where they are located. Creating a new `$P0 = new 'Foo'`
could give you two completely different types of objects if Foo were defined
differently on two or more threads. We could use STM or some other system to
allow data sharing, but how do we know if we need to share in the first place?

Do we use the nuclear approach and insert STM into every single memory
transaction in the system? If we do, we certainly gain some level of ensured
consistency but at the added cost of a transaction occuring for every single
memory access, not just the cross-thread ones. We could try to narrow that
down by including a series of flags and pointers. Each PMC would have to
contain a flag to say whether it was being actively shared. If shared, we
can selectively employ STM, otherwise we can avoid it. But having a shared
PMC like that creates huge implications in terms of GC at least. Also we
would need to determine what to share and when. If we have a shared array,
all the children of that array must also be marked as shared. However, if we
remove an item from that array, we can't necessarily unshare it and reclaim
the performance hit of transactions for that object.

In any case, this isn't the problem of Parrot core. It's not our job to impose
STM on all programs and all HLLs, especially if they all don't want or need
it. This is especially true if we can't make guarantees about the performance
implications of it.

STM is only one example of a system we could use to share data safely, but it
does make my point pretty well.  Any other mechanism would suffer the same
drawbacks.


