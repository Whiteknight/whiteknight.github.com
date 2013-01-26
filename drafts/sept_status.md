---
layout: post
categories: [Parrot, Personal]
title: September Status
---

First, some personal status:

### Personal Status

I haven't blogged in a little while, and there's a few reasons for that. I'll
list them quickly:

1. Work has been...tedious lately and when I come home I find that I want to
   spend much less time looking at a computer, especially any computer that
   brings more stress into my life. Also,
2. My computer at home generates a huge amount of stress. In addition to
   several physical problems with it, and the fact that I effectively do not
   have a working mouse (the built-in trackpad is extremely faulty, and the
   external USB mouse I had been using is now broken and the computer won't
   even book if it's plugged into the port), I've been having some software
   problems with lightdm and xserver crashing and needing to be restarted
   much more frequently than I think should be needed. We are planning to buy
   me a new one, but the budget won't allow that until closer to xmas.
3. The `io_cleanup1` work took much longer than I had anticipated. I wrote a
   lot more posts about that branch than I ever published, and the ones I did
   publish were extremely repetitive ("It's almost finished, any day now!").
   Posting less means I got out of the habit of posting, which is a hard habit
   to be in and does require some effort.

I'm going to do what I can to post something of a general Parrot update here,
and hopefully I can get back in the habit of posting a little bit more
regularly again.

### `io_cleanup1` Status

`io_cleanup1` did indeed merge with almost no problems reported at all.
I'm very happy about that work, and am looking forward to pushing the IO
subsystem to the next level. Before I started `io_cleanup1`, I had some plans
in mind for new features and capabilities I wanted to add to the VM. However,
I quickly realized that the house had some structural problems to deal with
before I could slap a new coat of paint on the walls. The structure is, I now
believe, much better. I've still got that paint in the closet and eventually
I'm going to throw it on the walls.

The `io_cleanup` branch did take a lot of time and energy, much more than I
initially expected. But, it's over now and I'm happy with the results so now I
can start looking on to the next project on my list.

### Threads Status

Threads is very very close to being mergable. I've said that before and I'm sure
I'll have occasion to say it again. However there's one remaining
problem pointed out by tadzik, and if my diagnosis is correct it's a doozie.

The basic threads system, which I outlined in a series of blog posts ages ago
goes like this: We cut out the need to have (most) locks, and therefore we cut
out many possibilities of deadlock, by making objects writable only from the
thread that owns them. Other threads can have nearly unfettered read access,
but writes require sending a message to the owner thread to perform the update
in a synchronized, orderly manner.  By limiting cross-thread writes, we cut
out many expensive mechanisms that would need to be used for writing data,
like Software Transactional Memory (STM) and locks (and, therefore, associated
deadlocks). It's a system inspired closely by things like Erlang and some
functional languages, although I'm not sure there's any real prior art for the
specifics of it. Maybe that's because other people know it won't work right.
The only thing we can do is see how it works.

The way nine implemented this system is to setup a Proxy type which intercepts
and dispatches read/write requests as appropriate. When we pass a PMC from one
thread to another, we instead create and pass a Proxy to it. Every read on
the proxy redirects immediately to a read on the original target PMC. Every
write causes a task to dispatch to the owner thread of the target PMC with
update logic.

Here's some example code, adapted from the example tadzik had, which
fails on the threads branch:

    function main[main](var args) {
        var x = 1;
        var t = new 'Task'(function() { x++; say(x); });
        ${ schedule t };
        ${ wait t };
        say("Done!");
    }

Running this code on the threads branch creates anything from an assertion
failure to a segfault. Why?

This example creates a closure and schedules that closure as a task. The task
scheduler assigns that task to the next open thread in the pool. Since it's
dispatching the Task on a new thread, all the data is proxied. Instead of
passing a reference to Integer PMC `x`, we're passing a `Proxy` PMC, which
points to `x`. This part works as expected.

When we invoke a closure, we update the context to point to the "outer"
context, so that lexical variables ("x", in this case) can be looked up
correctly. However, instead of having an outer which is a `CallContext` PMC,
we have a `Proxy` to a `CallContext`.

An overarching problem with `CallContext` is that they get used, a lot. Every
single register access, and almost all opcodes access at least one register,
goes through the CallContext. Lexical information is looked up through the
CallContext. Backtrace information is looked up in the CallContext. A few
other things are looked up there as well. In short, CallContexts are accessed
quite a lot.

Because they are accessed so much, CallContexts ARE NOT dealt with through the
normal VTABLE mechanism. Adding in an indirect function call for every single
register access would be a huge performance burden. So, instead of doing that,
we poke into the data directly and use the raw data pointers to get (and to
cache) the things we need.

And there's the rub. For performance we need to be able to poke into a
CallContext directly, but for threads we need to pass a Proxy instead of a
CallContext. And the pointers for Proxy are not the same as the pointers for
CallContext. See the problem?

I identified this issue earlier in the week and have been thinking it over for
a few days. I'm not sure I've found a workable solution yet. At least, I
haven't found a solution that wouldn't impose some limitations on semantics.

For instance, in the code example above, the implicit expectation is that the
x variable lives on the main thread, but is updated on the second thread. And
those updates should be reflected back on main after the `wait` opcode.

The solution I think I have is to create a new dummy CallContext that would
pass requests off to the Proxied LexPad. I'm not sure about some of the
individual details, but overall I think this solution should solve our biggest
problem. I'll probably play with that this weekend and see if I can finally
get this branch ready to merge.

### Other Status

rurban has been doing some great cleanup work with native PBC, something that
he's been working on (and fighting to work on) for a long time. I'd really
love to see more work done in this area in the future, because there are so
many more opportunities for compatibility and interoperability at the bytecode
level that we aren't exploiting yet.

Things have otherwise been a little bit slow lately, but between
`io_cleanup1`, `threads` and rurban's pbc work, we're still making some
pretty decent progress on some pretty important areas. If we can get threads
fixed and merged soon, I'll be on to the next project in the list.




