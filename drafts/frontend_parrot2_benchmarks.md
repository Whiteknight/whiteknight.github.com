I've been working on a branch called `whiteknight/frontend_parrot2` recently.
The purpose of this branch, as I have discussed in a few other previous blog
posts, is to rewrite the Parrot frontend executable to bootstrap into PIR
earlier, and do more work from the runloop. The benefits to this strategy are
a reduced number of embedding API calls, and no reliance on nested runloops
for executing `:init` functions as soon as the program starts. The new system
boostraps into a short PIR driver program called `prt0.pbc`. This short driver
program currently does very little except process a handful of command-line
arguments and execute the user program. In the future it could do more such as
defining object types and providing a small common runtime.

Today I've put together a few simple benchmarks to show the new system in
action. These benchmarks do confirm that the new frontend is faster, but only
really for cases where we have a lot of `:init` subs to execute. In other
cases it is competitive with the current frontend, or even a little slower.
I suspect strongly that there are some optimization opportunities available
which I haven't made good use of yet. Most of my work so far has been trying
to get this system working, not trying to tune and optimize it.

The test program I am using, `test.pir` is a simple program. It contains the
following snippet, copy+pasted a certain number of times:

    .sub '' :init :anon
        noop
    .end

Basically, what I'm testing for is dispatch performance for `:init` subs, not
any kind of execution performance or other details. I'm also running this
program from precompiled .pbc, so I am cutting out performance issues related
to IMCC. For this simple program, these effects are not significant. However,
I am trying to isolate them away in any case. For the benchmark, I use a
simple shell script to execute this program 1000 times in a loop, and time
the results.

For one instance of an `:init` sub, here are some representative timings. The
three sets of numbers are for the old frontend, the new frontend, and
miniparrot. Miniparrot is, of course, just like the old frontend except it
doesn't load the config hash on startup. This change is small but noticable
in these benchmarks and worth keeping in mind:

real	0m12.885s
user	0m15.430s
sys	0m7.280s

real	0m12.954s
user	0m16.050s
sys	0m7.890s

real	0m12.240s
user	0m14.760s
sys	0m6.790s


Here's the same test run, with 100 instances of the `:init` function:

real	0m13.293s
user	0m18.220s
sys	0m8.950s

real	0m13.789s
user	0m20.080s
sys	0m10.200s

real	0m12.956s
user	0m16.660s
sys	0m7.270s


And here is the same test with 1000 instances:

real	0m21.293s
user	0m21.860s
sys	0m8.830s

real	0m20.337s
user	0m22.360s
sys	0m8.110s

real	0m21.017s
user	0m21.340s
sys	0m9.240s

For the 1 and 100 runs, the old frontend does outperform the new one. It's not
a huge difference, only about 1-3% in startup time. For large programs or
libraries with a thousand `:init` functions or more, the scales do start to
tip in favor of the new system. What's very interesting is that for the case
of 1000 `:init` subs, the new frontend even out-performs miniparrot.

The new system is not a huge performance boost across the board. Part of me
naively hoped that it would be, but it isn't right now. I do have more work
to do in terms of optimizing the system, so I am hopeful that I can squeeze
some small gains out from that.
