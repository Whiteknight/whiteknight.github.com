Here's an example winxed program I wrote to benchmark some dispatch
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
uses all integer compares, and the internals of the loop is a single add
instruction between integers. This loop probably does not invoke GC even once,
and executes entirely in a single context. Run took just under two tenths of
a second to run, which is probably dominated by opcode dispatch costs and
register lookup costs (neither of which are huge).

The second test, using vtables on a built-in type takes about 135% more time
than the baseline. This isn't a huge jump, but now instead of just adding
a constant we are calling a vtable to fetch and return the value of the
constant. This loop still probably has no GC costs, no PCC costs, and is
dominated by opcode and vtable dispatch costs. Built-in vtable dispatch costs
are higher than the costs of adding a constant, as we would expect, but we
aren't going overboard here.

The third test uses method calls on PIR-defined objects to fetch the constant
values. Here, the performance dropoff is noticable: a whopping 2204% longer
than the baseline, and 1623% longer than C-based vtable dispatch. Of course,
these costs aren't unexpected. We now have to include PCC costs for calling
the method, and GC costs to help reclaim the contexts and signatures and
things. I've said before (and I will say several hundred more times on this
blog and elsewhere) that we can decrease PCC costs significantly, but this is
what we have right now.

Now here's where the results get surprising. What happens when we call a
vtable override? Vtable overrides are just Sub PMCs, the same as methods, and
use PCC dispatch and GC just the same. However, there is a huge difference in
timings. vtable overrides take 200% longer in this benchmark compared to
ordinary methods, and a gigantic 3259% longer than ordinary vtable calls. In
other words, one vtable override on a built-in type can take one amount of
time, and the same exact operation on a subclass which overrides that vtable
will take over 3200% longer.

Why?

The answer is actually pretty simple. In addition to the PCC and GC costs of
the dispatch mechanisms, vtable overrides must recurse and create a new
runloop. Creating the new runloop to execute the override, setting up the
execution environment for it, executing it through PCC, and returning a value
back up the chain is what takes all that time. Those nested runloops are
nefarious, which is one reason why we need Lorito so much. If we don't have
to recurse runloops for vtable overrides, we can cut the worst-case dispatch
time by half.

Of course, Lorito could be a long way off depending on a number of factors.
One thing you can do right now to make your own code faster is to avoid vtable
overrides entirely, and use methods where possible. As part of my 6model
plans, we can reduce the total number of vtables, and maybe add a mechanism
to execute vtable overrides in-line like methods, if we are calling them from
the runloop (and not from nested C code).


