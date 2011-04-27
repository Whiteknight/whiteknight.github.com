A few days ago I published a very long post about my
[vision for threading][threading_vision]. Today I'm going to post some
follow-up information to try and tie some of the ideas together.

In addition to writing long, rambling blog posts I am also an avid reader. I
read a heck of a lot of interesting information about technologies which I use
but also technologies which I do not use. I really like seeing how the other
half operates, especially if I can learn some lessons to bring back and
improve my own work. Recently I've been introduced to a variety of new ideas
and I realized that several of them really tie in to my ideas for Parrot. What
I didn't realize when I wrote my last post is that some of the things I was
describing have actual names and terminology associated with them. I'm going
to use some of that new terminology to describe some details of my concurrency
ideas more succinctly.

We have a system with two threads running in parallel, sharing data between
themselves. There are no good, general-purpose ways for one thread to know
precisely where in the flow of execution the other is without stopping one of
the two at a checkpoint. Maybe the "wait here until I am ready for you"
mechanism is a good one in some situations, but it's not good in all of them.
On a single core processor, this becomes a non-issue because while one thread
is waiting, the other thread can be chugging along at full speed. However, in
a multicore environment, suddenly one of your cores is sitting idle waiting
for the rest of the system to catch up. Even if we want to allow this kind of
waiting mechanism in Parrot, I see no particular reason why we would make it
the default synchronization mechanism. Telling one thread to stop and do
nothing for an unspecified amount of time is very wasteful. As an aside, we
could use those pauses to run background behaviors like GC and not have it be
a total waste, but that's besides the point. We only have so many such
internal tasks to go around. If you think we can fill empty execution cycles
with GC in a two-threaded system, think about a system with 100 threads and
suddenly you start to see that there is going to be a huge amount of waste.

Back to topic. There's no good way, without pauses, to know precisely where
one thread is at any given time. What we can do is query the thread to see if
it has passed certain checkpoints, but we don't know how far past that
checkpoint the thread has gone. Let's look at a short example of two thread
functions sharing a single data item and using a flag to communicate ready
state:

    function thread_A(var sharedData) {
        ...
        sharedData.IsReady = 0;
        sharedData.Update();
        sharedData.IsReady = 1;
        ...
    }

    function thread_B(var sharedData) {
        ...
        var value = null;
        var snapshot = sharedData.GetSnapshot();
        if (snapshot.IsReady == 1)
            value = snapshot.GetValue();
        ...
    }

This is an extremely naive system which uses sentinels to maintain data
consistency. To alleviate some distracting concerns, I've added in a mechanism
here to take an instantaneous snapshot of the shared data so that we can
guarantee the `sharedData` is consistent between the time we test the flag
and the time we read the value. Other ways to acheive this kind of thing would
be with locks or critical sections, or whatever. We can spare those details
for now and leave everything up to the imagination of the programmer. My point
with this example is that if we ask "is `IsReady` set?", we can say that it
is or that it isn't, but we can't say how long since it has been or has not
been.

In a multi-core system the two threads can be executing in true parallel on
two different cores, executing at their own speeds because of memory, disk,
and cache timing issues. There's no way to even count instructions and say
that the two threads will be in certain positions after X instructions have
been executed. Because of a variety of issues, machine code instructions do
not even execute in the same amount of time. I'm going on about this far more
than I probably should be, but my point is a simple one: We can't make
guarantees about where two threads are in relation to one another, so it
doesn't make a hell of a lot of sense that we would try to guarantee immediate
consistency between data in those threads.

If we set a piece of data in one thread, we have absolutely no way to prove
exactly when that data will be available to the next thread. The two threads
may be running on a single core machine and the second one might not even
start executing again for another couple milliseconds. Or the two threads
could be on different cores and delays in cache and memory flushes could
prevent the update from being seen even if the read on the second thread
happens directly after the write from the first thread. Or, the second thread
may be executing some other bit of code entirely, and not even attempt to read
the updated value any time soon.

If we understand and embrace this kind of system, we start to understand that
what we want is *[eventual consistency][eventual_consistency]*. We can
approximate real-time consistency if we hate ourselves and are willing to jump
through hoops and subject users to locks and pauses. Maybe our HLLs can do
that kind of stuff, but Parrot should not be doing it internally.

If we know that we aren't immediately consistent, if two threads don't know
what the other thread is doing, if the second thread doesn't know whether the
first thread has written data and should therefore be expecting to read
updated data at all, and if the first thread has written data but doesn't
know whether the second thread is even prepared to read it, we can start to
embrace the idea of eventual consistency between threads.

Of course, this does raise some issues that can be a little bit weird by
default. Consider this next code example which uses the concurrency proxies
I've talked about in my previous post, and an API for them that I am inventing
hastily:

    function thread_A() {
        var data = GetComplexData();
        var b = ThreadScheduler.GetThread("thread_B");
        b.SetValue(0);
        b.share_safe("data", data);
        var this_thread = ThreadScheduler.CurrentThread();
        while (1) {
            // This automatically applies internal data updates received.
            this_thread.CheckMessages();
        }
    }

    function thread_B() {
        var this_thread = ThreadScheduler.CurrentThread();
        this_thread.CheckMessages();
        var data = ThreadScheduler.GetMessageData("data");
        // "data" here is a concurrency proxy
        int d1 = data.GetValue(); // should be 0
        data.SetValue(5);
        int d2 = data.GetValue(); // might still be 0!
    }

So in this example, thread A creates a local data item and shares it with
thread B. Thread B receives a proxy to that data item, reads the current
value, updates it to a new value, then reads from it again. Behind the scenes
we know that the update operation is a complex one. We can't just write data
between threads, we need to synchronize and do other stuff to prevent data
corruption. In the basic system, I suggested the update should send a message
instead of performing the update directly. That message would be received and
processed by the message loop I set up in thread A. Here's the catch: If the
message sent is done synchronously and thread B pauses while the update
happens, it's no big deal. We lose a little performance but we have data
consistency and the second read in thread B returns "5" as we expect. However,
if the message is passed *asynchronously*, it is unlikely that the data on
thread A would have been updated by the time thread B tries to read it again,
and the second read could easily return "0" instead.

Some people will look at this and suggest that the asynchronous case is
absolutely, positively unacceptable. If we have 10 threads all writing and
reading data to a single shared global data item, and they are getting
incorrect state information the whole system could be doing bad things! And
if it's a problem with 10 threads, it's certainly a problem with 100 of them,
an even bigger problem at that.

There are a few solutions to this dilemma. The first, of course, is to pause
and wait. Use the synchronous message solution. Here's an example:

    function thread_B() {
        var this_thread = ThreadScheduler.CurrentThread();
        this_thread.CheckMessages();
        var data = ThreadScheduler.GetMessageData("data");
        // "data" here is a concurrency proxy
        int d1 = data.GetValue(); // should be 0
        data.SetValue(5);
        this_thread.DeliverAllMessages();
        int d2 = data.GetValue();
    }

In this, I added the call to `this_thread.DeliverAllMessages()`, which would
block until all sent messages are received. This will probably require the use
of some kind of receipting system, but that's not a big deal.

The second solution to the dilemma might be to ask the proxy whether it has
been updated yet:

    function thread_B() {
        var this_thread = ThreadScheduler.CurrentThread();
        this_thread.CheckMessages();
        var data = ThreadScheduler.GetMessageData("data");
        // "data" here is a concurrency proxy
        int d1 = data.GetValue(); // should be 0
        data.SetValue(5);
        while (1) {
            int d2;
            if (ThreadScheduler.IsProxyUpdated(data)) {
                d2 = data.GetValue();
                break;
            } else
                ...
        }
    }

In this case we ask whether the proxy is in a consistent state (has no unread
messages in its queue), and spends cycles doing some other task if it isn't.
When we exit that while loop, we know that the value d2 will be "5" as we
expect. A third option is to realize that Tasks in this system are cheap to
create. They are just continuations and Parrot makes and uses continuations
all the time. We can send the update message along with a callback, realizing
that messages sent will be checked in order. If that callback launches a new
Task on thread B, we get a similar behavior to the synchronous case:

    function thread_B() {
        var this_thread = ThreadScheduler.CurrentThread();
        var thread_A = ThreadScheduler.GetThread("thread_A");
        this_thread.CheckMessages();
        var data = ThreadScheduler.GetMessageData("data");
        // "data" here is a concurrency proxy
        int d1 = data.GetValue(); // should be 0
        data.SetValue(5);
        var next_task = function(int d2) {
            // here, d2 = 5.
            // Also, this closure is able to access all the lexically-scoped
            // variables from the enclosing function.
        }
        thread_A.SendCallbackMessage(
            function(B, task) {
                var A = ThreadScheduler.CurrentThread();
                var data = A.GetLocalData("data");
                B.ScheduleTask(task, data.GetValue());
            },
            this_thread,
            next_task
        );
        return; // end the task
    }

What does this system do? The `thread_A.SendCallbackMessage` function sends
a callback function to be executed on thread A. Everything inside that
function is executing in the context of thread A. That callback then gets a
reference to thread B and schedules a new task for it. That new task gets the
updated version of the shared data, and executes as a closure in the lexical
environment of the thread_B function, after that function has exited. Notice
that instead of the `new_task` callback being a function, we could have made
it a continuation to a label in the `thread_B` function itself, and jumped
right back to the current lexical scope after the message has been processed,
or we could make `thread_B` a coroutine. At the bottom we `yield` instead of
return, then when we call `thread_B` again we continue executing from where
we left off, or ....

In short, we have lots of interesting possibilities here. If you think about
a thread as being an execution area for a sequence of continuations, we start
to come to a system which is extremely powerful and flexible.

That said, there are also plenty of places where we can get into trouble. In
the last code example I wrote, I could have easily and accidentally accessed
a variable which was created on thread B and lexically scoped there. When I
attempted to access it from thread A, I would be using a reference to the
variable itself, not executing through a proxy. This would lead to some
serious and hard-to-debug problems. One solution there is to check the current
thread in the lexical variable lookup, and throw a fit (or, just an exception)
if we attempt to look up a lexical variable defined on a different thread.
Since lexical lookups search for the variable up the context chain, and since
contexts would be tied to a particular thread, we could easily check if
`current_thread == context.thread` and throw an exception if not. It is
actually very important that the context contains a reference to the thread
it lives in, so that when we do GC we don't prematurely collect the thread
object without also collecting the contexts which think they are running on
that thread. That's a different topic for another time.





