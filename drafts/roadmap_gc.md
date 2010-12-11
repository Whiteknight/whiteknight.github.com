Parrot needs a new Garbage Collector. We need it yesterday. Hell, we need it
[three years ago][gc_gsoc]. Many people have tried and failed (myself
included) to fix our memory-management woes, but I think that we finally
have enough people who are willing and able to tackle it to get the problem
solved.

[gc_gsoc]: /2009/05/06/switching_gears_gc.html

Parrot's resident magical coding robot, bacek, has been putting together a
design for a new GC algorithm that's based on a generational approach. This is
going to be a major part of the puzzle but it isn't really enough for us to
be able to declare victory over our memory-management problems. GC performance
is really based on two factors: the efficiency of the algorithm (throughput)
and the number of objects we need to manage (pressure).

If we have a really great algorithm (high throughput) but we have a bajillion
and a half objects (high pressure) we're still going to have lousy
performance.

Parrot puts far too much pressure on it's GC. We create too many long-lived
objects which each need to be marked on every single GC iteration, and we also
create too many short-lived objects which get created and then collected
very quickly.
