I've written two posts about my idea for concurrency in threading. Since this
is only my suggestion and not the accepted, "official" solution by any means
that certainly seems like plenty. There's no sense constantly talking about an
idea like this until I get a green light from the rest of the community to
start pursuing this path in earnest. I do have one more thing to talk about
with this idea before I lay it to rest completely. I want to talk today about
how we actually move from where we are now to where I suggest we should be
going. The first two posts talked about the idea itself and some of its
merits, while this post will lay out a basic, though incomplete, method to
pursue it. Once we decide as a community exactly what kind of concurrency
solution we would actually like to see in Parrot, we can start filling in the
details that have been omitted here.

The first step in moving towards the new threading system is to start
refactoring some of the current datastructures and concepts that form the
underpinning of the whole thing. Right now in Parrot we have an old and broken
threading system which is both unusable now for any but the most trivial tasks
and is also considered by several people to be an undesirable design for our
long-term needs. The absolutely first step we must be willing to take if we
want to get anywhere is to list the entire threading system as deprecated.
This includes all the threading primitives and the current threading API (if
you can call such a thing an "API" at all). We need to deprecate the
threading-related and task-related PMCs: ParrotThread, Task, Timer,
ParrotInterpreter, Scheduler, SchedulerMessage, and maybe a few other things
that I haven't considered yet. Some of these PMC types need to disappear
entirely if we want to make way for the new system. Some of these can hang
around, so long as we are able to start changing some of the necessary parts
of the interface. Anything threading, scheduling, or task related needs to
be open for modification. I suspect that we can do most of the necessary work
without impacting PCC, Exceptions, or Continuations at all. Internally,
Exceptions might need to refactors but those changes shouldn't impact the
user at all.

The second step is a tricky one, but I think we can piggy-back on the work the
Lorito team is doing. What we need to do is separate out the major data bits
necessary for "global" items. Currently we have two real data items: The
interpreter and the Context. Threads are all individual interpreters, which
creates some of the major problems in the current system. Instead, I think we
need to start separating things out to represent specific responsibilities. I
see the need to have four objects instead of the current two:

1. **The Interpreter**. There is one interpreter per process. It represents a
   relatively small subset of "global" data for the process. The interpreter
   will contain things like the GC (if we use a global GC instead of a series
   of thread-local mini GCs), A list of global data items (immutable
   packfiles, opcode libraries and the opcode lookup hash, loaded libraries,
   global vtables for built-in types, hash seed, the root namespace, and other
   flags and global settings), a pointer to the current "master" thread, and
   very little else.
2. **Thread**. The Thread will take the place of the interpreter throughout
   much of the system. The thread represents the thing that is actually
   executing, and contains information needed for current execution: The
   current packfile information, the current context, the current HLL,
   flags, options, and limits relating to the thread and the tasks that run on
   it, etc. The thread will also contain a scheduler of sorts which will
   contain a list of tasks and the logic necessary to switch between them.
   Every thread will have a reference to the interpreter, although only the
   "master" thread has the live reference for direct access. All other threads
   will contain a proxy reference, or an API for accessing the data fields of
   the interp.
3. **Task**. The Task is the actual thing to execute. Multiple tasks can live
   together and execute sequentially on a single thread. The Task contains
   information about the current executing code: The current context, the
   current continuation needed for starting the task, the arguments to the
   task, etc. When the Thread launches a new Task, the information about
   current packfile and context are read from the task and inserted into the
   thread for reference, the current execution location of the current task
   is saved in a continuation, and the continuation of the next task is
   invoked.
4. **Context** The context represents the current execution frame, one per
   sub call. This contains most of the information that Contexts contain
   now: A link to caller and outer contexts, a list of current exception
   handlers, the current array of registers, the current lexpad, a pointer
   to the currently executing Sub, the arguments to that sub, the return
   continuation, the current PC pointer, pointer to the current set of
   constants, etc. The Context doesn't have to change much from what it is
   currently.

The basic system is this: There is one interp per process, which delegates
execution to many Threads. Each Thread maintains a collection of Tasks, which
are individual flows of execution. Each Task then contains a graph of Contexts
to help manage the actual execution. This basically adds two new intermediate
objects to the current flow: We have an interp (because we don't really allow
"real" threading right now), and it points directly to the graph of contexts.

For this step, the `Interp*` that we know now needs to be broken up into two
parts: The "global" interp, and the "local" Thread. GSoC student Chandon
recommended this very course of action last summer. Once we have that, most
function calls inside Parrot will probably need to be updated to take a
pointer to the current executing Thread instead of taking a pointer to the
global interp. This represents one of the single biggest challenges, and is
itself going to require a major deprecation of the extending and embedding
APIs. Once we hit this milestone, however, disruptions to the user can be
kept to a minimum for the rest of the process.

After updating the APIs to use Thread instead of Interp, we can later
introduce the Task objects. Tasks are just a tuple of current Context, current
args, and current Continuation, so it should not be too difficult to add in
a Task object to the Thread object to help keep this important data straight.

Step three is the advent of green threads. Green threads are basically just
Tasks that execute subsequently on a single Thread. To come to this step we
need to add a TaskScheduler mechanism to the Thread, create a mechanism to
switch between Tasks, and create a new API for scheduling and managing Tasks.
The first prototype will probably be a cooperative system like what was used
back in the olden days of computing: Tasks must be manually switched by API
calls from the user. Once we get a few details sorted out in OS-land, we can
then implement pre-emptive switching like modern users expect. This is
going to require some modifications to the runloop among other things to
ensure we only switch tasks at an acceptable opcode boundary. We're also going
to have to disable thread switching if we are inside a nested runloop, for
a variety of reasons. If the Lorito team can succeed in disallowing nested
runloops eventually, this limitation goes away.

Step four is to start implementing the data sharing mechanisms. This is an
internal-only change, and will likely sit unused or under-used until we put
on the final touches in the next step. What we can do is implement the
necessary opcodes, PMCs, and API functions and begin writing tests for them.

The final big piece in the puzzle is to extend the system to the true hybrid
approach: We need to add in the infrastructure to create and manage multiple
Threads. This is going to require additions to the embedding and extending
APIs, the creation of a new Thread management API (and maybe the creation of
an interpreter-global ThreadScheduler PMC), and a few other things. This is
also where we are going to have to finally start looking hard at all the
inequities and infelicities of threading implementations across OSes, and
writing up the necessary abstractions to allow Parrot's Threads to run on all
our supported platforms.

   