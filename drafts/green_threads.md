If you haven't been keeping up with the #parrot hubub, bub, you might not know
that there are some new features coming down the pipe in the world of 
concurrency.

Two summers ago, GSOC student Chandon was working on a project to add hybrid
threading to Parrot. I won't take the time to really explain what that means
because I've talked about that kind of system, at length, in previous posts.
Long story short, Chandon made a valiant effort but didn't get his project
finished by the end of the summer. He got busy with school and other stuff
and I was distracted by other pressing issues, and so his branch languished
for a while.

Languish no more! Parrot newcomer nine decided that Parrot really needs
concurrency, and he is just the person to do it. After we ripped out the old
threading system, nine got to work on the old `gsoc_thread` branch that
Chandon left behind. He updated it to master fixing conflicts along the way
and fixed it up so that we had working green threads. We have a few small
fixes that still need to be made on the branch, but it's very possible that
we can get it merged into Parrot next week following the release.

Parrot is going to have green threads soon. What does that mean for us?

Green threads are not like real OS-level threads. In Parrot parliance, they
are going to be called "Tasks", and the `Task` PMC type will represent a
green thread. Internally Parrot is still a single-threaded system, but now we
have the ability to schedule preemptive tasks, and let the Parrot scheduler
manage them and switch them out transparently.

At first, this isn't going to do much. There are some limitations because of
implementation details in the current system. One such limitation is that
green threads cannot be switched during execution of a nested runloop. They
can only be switched from inside the top-most runloop. This is a problem that
will magically disappear with the advent of M0, but that could be a long
way away.

What green threads do for us is that they serve as a stepping stone to start
getting to new features. Asynchronous IO (AIO), for instance, is something
that is suddenly well within reach. The IO subsystem can be modified to make
asynchronous calls at the OS level and block the current task only. This means
that the scheduler can be executing other tasks while the IO request is
pending in the same thread. With relatively small additions, the scheduler
can be upgraded to check for completed IO events and take appropriate action.

One other thing we can start working on now are mailboxes and messages, or at
least the archetypes for them. Getting interfaces for these things into place
now means that we can get feedback on those interfaces and start writing code
for them now. If you use mailboxes to pass messages back and forth between
tasks now, eventually those same exact interfaces will be used to seamlessly
move data between separate threads as well, without user code needing to be
updated much (if at all). That's exciting to think about.

We can start talking about working with signals, events, timers, and other
things which Parrot has had poor or no support for previously. Asynchronous
callbacks from external libraries also can start taking on a new life and
maybe be upgraded to work as tasks. 

Last but not least, having green threads around means that we can start the
hard work of adding in OS-level threads and working towards the ultimate
goal of a hybrid threading system. That's a huge project and I don't want to
make any promises about timelines, but the work can start in earnest now and
that's fun to think about.

Green threads by themselves don't bring much to the party. However, it's the
things that we can build on top of them, and the things that we can do in
their presence that make this such a fun new addition to Parrot. 
