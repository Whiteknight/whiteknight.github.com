---
layout: post
categories: [Rosella, Event]
title: Rosella Event Rewrite
---

I've had an [Event library][event_post] as part of Rosella almost since that
project began. I was inspired by a few other publish/subscribe systems I had
been working with at the time and put together a relatively simple variant for
Parrot. However, while I've been pretty happy with other things I've done in
Rosella, I have not been happy with the Event library. I rewrote it a few times,
but could never really get it working the way I wanted it to work. I let it sit
for a while and focused on other projects instead.

[event_post]: /2011/04/21/rosella_event.html

Now, with the advent of Green Threads in Parrot, I've decided to go back and
give the Event library a second look. Anything I can do to utilize and
exercise the new green threads functionality will be a plus, and it finally
gives me real motivation to go back and do eventing the right way. As of
this week I have rewritten the library and have the basics of it available for
use.

In the most simple usage, you create an Event object with a callback:

    var event = new Rosella.Event();
    event.subscribe("test subscriber", function(p) { ... }, "immediate");
    event.publish(...<args>...);

Each subscriber gets a string name and a callback. The third argument to the
subscribe method is the name of the dispatcher to use. The immediate
dispatcher executes the callback immediately in line during the publish
method. The "task" dispatcher schedules the callback to execute as a green
thread. Also available, but not currently implemented, is a "thread"
dispatcher which will dispatch the callback to a new worker thread. Once
Parrot has thread support added (and we're not too far off now), I'll finish
up that last option.

Here is a short example:

    var event = new Rosella.Event();
    event.subscribe("foo", function(p) { say("in foo"); }, "task");
    event.subscribe("bar", function(p) { say("in bar"); }, "immediate");
    event.publish();
    say("done");
    sleep(0.2);

...And the output, on my system:

    in bar
    done
    in foo

So that's not too shabby, and it's a really easy way to use Parrot's new green
threads from the safety of Winxed and Rosella. There are a few more details I
want to iron out still, and the library needs to be a lot more flexible to
really fit in with the rest of the Rosella fold. I suspect in a few days I'll
declare the rewrite to be a stable part of Rosella.

In related news, hacker **nine** is still hard at work on the new threading
implementation. I've been following along with some of his work and am
starting to get pretty excited about it. He's been following some of my
[designs][threads_post_1] and [ideas][threads_post_2] pretty closely, and it's
all actually (surprisingly) working out for him. Whether he sticks with that
design or needs to improvise to get passed some road blocks is for him to decide
as he progresses.

[threads_post_1]: /2011/04/23/vision_parrot_concurrency.html
[threads_post_2]: /2011/04/29/vision_parrot_concurrency_2.html

I've done a little work trying to port the green threads implementation to
work on windows, but am hampered because I don't have a windows machine
available to do any serious testing or developing on. My procedure right now
is to write code at home on my laptop and try to test it on my windows machine
at work. As you can imagine, for somebody as accustomed to a rapid iterative
development cycle as myself, that kind of progress is slow and stultifying.
If I do end up getting a new computer sometime soon, I might be able to pick
up some speed. I hope to have green threads working on windows before the
3.11 release, but can't make any promises considering my current development
velocity on that project.

I have a few other ideas of things to add to Rosella to facilitate tasks and
threading, but I think I am going to wait until Parrot has threads to really
pursue anything else. Right now, it's cool enough to have green threads
working, and have an eventing library that is worth using.
