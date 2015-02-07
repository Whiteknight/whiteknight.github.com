---
layout: post
categories: [Parrot, 6model]
title: Timings for Vtable Overrides
---

Following my [6model post from a few days ago][6model_post], I wanted to show
a little bit more about some of the problems with the current Parrot object
model. I wanted to show the performance disparity between built-in vtables and
user-defined vtable overrides. The built-in versions are cheap to call, and I
knew that overrides would be more expensive. I wasn't aware how much more
expensive, so I put together a benchmark to compare some things.

[6model_post]: /2011/05/07/6model_on_parrot.html

On a side note, several Rakudo developers are showing that there have been
some significant performance regressions since the 3.0 release. We've spent a
few days looking into causes for that, at least some of which may be related
to GC tunings. This morning bacek ran valgrind on the Rakudo build, and showed
surprisingly that GC only accounts for about 12% of total execution time,
which is actually not a bad result. What is surprising is that PCC, especially
the `fill_params` function which I will talk about below, eats up just as
much time. What stinks is that the two synergize: PCC creates PMCs, which
causes additional churn in GC. If we make PCC more efficient, we cut down on
GC pressure along the way. Today I'm going to talk about one special case
where PCC really fails hard. I'm preparing subsequent posts to talk about
possible solutions, and I will post them later this week.

Here's an example winxed program I wrote to benchmark some common dispatch
mechanisms:

    class MyTestClass {
        function get_integer[vtable]() { return 1; }
        function get_int() { return 1; }
    }

    function main[main]() {
        var total_runs = 1000000; // one million times
        var obj = new MyTestClass;
        var myint = new 'Integer';
        count_time("no dispatch (base line)", function() {
            int count = total_runs;
            int result = 0;
            for (int i = 0; i < count; i++)
                result = result + 1;
        });
        count_time("vtable calls", function() {
            int count = total_runs;
            myint = 1;
            int result = 0;
            for (int i = 0; i < count; i++)
                result = result + myint;
        });
        count_time("method calls", function() {
            int count = total_runs;
            int result = 0;
            for (int i = 0; i < count; i++)
                result = result + obj.get_int();
        });
        count_time("vtable_override calls", function() {
            int count = total_runs;
            int result = 0;
            for (int i = 0; i < count; i++)
                result = result + obj;
        });
    }

    function count_time(string description, var code)
    {
        say(sprintf("Starting %s", [description]));
        float starttime = floattime();
        code();
        float endtime = floattime();
        say(sprintf("Total time: %fs\n", [endtime - starttime]));
    }

What this is doing is comparing three dispatch mechanisms: Regular vtable
calls, vtable override calls, and method calls. Which do you think is the
fastest? Which do you think is the slowest? Here are the results on my
machine:

    Starting no dispatch (base line)
    Total time: 0.189244s

    Starting vtable calls
    Total time: 0.257013s

    Starting method calls
    Total time: 4.172381s

    Starting vtable_override calls
    Total time: 8.376169s

The baseline test is entirely composed from integer operations. The for loop
uses all integer compares, and the internal op of the loop is a single add
instruction between integers. This loop probably does not invoke GC even once,
and executes entirely in a single context. It took just under two tenths of
a second to run, which is probably dominated by opcode dispatch costs and
register lookup costs (neither of which are huge). There isn't much in this
benchmark that we can optimize directly, unless we ran optimizers on it to
convert the 3-argument add with a constant to a single 2-argument in-place
add or even a 1-argument increment instruction. I'm not convinced that either
of those would actually be any faster, since the op dispatch costs and the
register lookup costs are still big pieces at this scale.

The second test, using vtables on a built-in type takes about 135% more time
than the baseline. This isn't a huge jump, but now instead of just adding
a constant we are calling a vtable to fetch and return the value of the
constant. This loop still probably has no GC costs, no PCC costs, and is
dominated by opcode and vtable dispatch costs. Built-in vtable dispatch costs
are higher than the costs of adding a constant, as we would expect, but we
aren't going overboard here. Again, I don't think there are many opportunities
to optimize here, so this number is probably as good as we are going to get.

The third test uses method calls on PIR-defined objects to fetch the constant
values. Here, the performance dropoff is noticeable: a whopping **2204% longer
than the baseline**, and **1623% longer than C-based vtable dispatch**. These
costs aren't unexpected, we know that a method call is more intensive than a
vtable call. We now have to include PCC costs for calling the method, and GC
costs to help reclaim the contexts and signatures and things. I've said before
(and I will say several hundred more times on this blog and elsewhere) that we
can eventually decrease PCC costs significantly, but this is what we have
right now. Each method call involves a find_method vtable call to fetch the
method object, then an invocation of that object through PCC. If we had a
system of polymorphic inline caching at the call site we could cache the
results of find_method and maybe save some time there. Also a JIT could do
some similar caching through type specialization. As we saw above, the
built-in vtable dispatch costs to call find_method aren't too severe anyway,
so things like JIT and PIC probably aren't the silver bullet we need to cut
out the bulk of the timing problems here. Since we aren't passing any
arguments to this method the brunt of the PCC costs in argument processing
can be avoided entirely.

Now here's where the results get surprising. What happens when we call a
vtable override to do exactly the same thing as in the previous examples?
Vtable overrides are just Sub PMCs, the same as methods, and use PCC dispatch
and GC just the same. However, there is a huge difference in timings. vtable
overrides take **200% longer** in this benchmark compared to ordinary methods,
and a gigantic **3259% longer than ordinary vtable calls**. In other words,
one vtable override on a built-in type can take one amount of time, and the
same exact operation on a subclass which overrides that vtable will take over
*3200% longer*. Since many vtable overrides are called from inside opcodes,
some ops will appear to take twice as long as a custom method to perform the
same exact operation.

Why?

The answer is actually pretty simple. In addition to the PCC and GC costs of
the dispatch mechanisms, vtable overrides must recurse and create a new
runloop. Creating the new runloop to execute the override, setting up the
execution environment for it, executing it through PCC, and returning a value
back up the chain is what takes all that time. Those nested runloops are
nefarious, which is one reason why we need Lorito so much. If we don't have
to recurse runloops for vtable overrides, we can cut the worst-case dispatch
time by half.

Also making this situation worse in perpetuity is that the current system of
vtable overrides are not nearly as amenable to things like PIC or JIT type
specialization optimizations in the future, because the vtable override is
found and executed internally. There is no vtable object to find and cache
anywhere prior to the lookup. These things might become possible after Lorito
and after 6model and a huge round of vtable refactors, but again that's not
even a portion of the problem. The biggest problems are the PCC dispatch
mechanism itself, and the argument processing mechanism.

Notice also that vtable overrides are not really special. Other types of
dispatch have the same performance characteristics as they do. For instance,
any time we call a PIR-defined method from C code (as happens in embedding
situations, for example), we also must recurse into a new runloop and will
have some of the same performance problems. Some add-on and other ecosystem
projects make heavy use of PCC invokes from C code, which would all be just
as bad as what I show here.

The solutions here are mostly clear. First off, if at all possible, avoid
vtable overrides. If you convert every vtable override in your system to a
method call instead, you will see some significant savings. Also, in the
future, Lorito should turn all these nested runloop calls into ordinary
subroutine dispatch calls, which in turn will make them all less expensive.
Of course, Lorito could be a long way off depending on a number of factors.
One thing you can do right now to make your own code faster is to avoid vtable
overrides entirely, and use methods where possible. As part of my 6model
plans, we can reduce the total number of vtables, and maybe add a mechanism
to execute vtable overrides in-line like methods, if we are calling them from
the runloop (and not from nested C code). I'll talk about all these plans in
future posts.


