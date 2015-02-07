---
layout: post
categories: [Parrot, Rosella]
title: Sorting
---

A few days ago I was reminded of an [old ticket about sorting][tt1356].

[tt1356]: http://trac.parrot.org/parrot/ticket/1356

We have 10 basic array types in Parrot:

    fixedbooleanarray.pmc
    fixedintegerarray.pmc
    fixedstringarray.pmc
    fixedfloatarray.pmc
    fixedpmcarray.pmc
    resizablebooleanarray.pmc
    resizableintegerarray.pmc
    resizablestringarray.pmc
    resizablefloatarray.pmc
    resizablepmcarray.pmc

I think this is absurd and we shouldn't have so many of these, but that's a
subject for a different post.

The big problem right now is that these array types don't have any common set
of inherited functionality, and only a few of these have a "sort" method for
sorting. The ticket I mentioned was to add "sort" method to the types where it
isn't implemented already and where it would make sense to have. The boolean
arrays don't need sorting really, but the other types do. Specifically, the
string arrays are lacking a sort routine while the integer and PMC types
already have it.

However, I feel like that's not a great solution. We shouldn't have to copy
and paste sort method code into all these types. We should either inherit the
code among the types which need it, or write up a sorting routine somewhere
else that can work on all array-like types. For a variety of reasons I don't
think inheritance is the way to go on this occasion because there isn't enough
common functionality between these array types that we could inherit easily.
Plus, I had a hunch that the performance was not so great on our current
quicksort implementation because it was written in C and used nested runloops
to execute the comparion routines. As my
[recent benchmarks][nested_benchmarks] showed, nested runloops are performance
killers.

[nested_benchmarks]: /2011/05/10/timings_vtable_overrides.html

As a quick sort of experiment, I implemented a new sort routine as part of
the new [Rosella Query library][query]. The initial implementation was naive,
but with a few easy optimizations I was able to produce some *interesting*
results. Here is my current quicksort implementation in winxed, as it appears
in the Rosella Query library:

[query]: /2011/05/24/rosella_query.html

    function qsort(var d, int s, int n, var cmp)
    {
        int last = n-1;
        while (last > s) {
            int pivot = s + int((n - s) / 2);
            int store = s;
            var tmp;

            var piv = d[pivot];
            d[pivot] = d[last];
            d[last] = piv;

            for(int ix = s; ix < last; ix++) {
                if (cmp(d[ix], piv) < 0) {
                    tmp = d[store];
                    d[store] = d[ix];
                    d[ix] = tmp;
                    store++;
                }
            }

            tmp = d[last];
            d[last] = d[store];
            d[store] = tmp;
            pivot = store;
            qsort(d, s, pivot, cmp);
            s = pivot + 1;
        }
    }

I inline all the pivot and swap code instead of breaking out into separate
functions for each, and I use loops to cut down on the amount of recursion
done. This is almost exactly the same algorithm as is used by the builtin
sort routine, so we are comparing apples to apples.

Here is an example of the benchmark code I've been using:

    function main[main]() {
        load_bytecode("rosella/benchmark.pbc");
        using Rosella.Benchmark.benchmark;
        load_bytecode("rosella/query.pbc");

        const int N = 100000;
        say(sprintf("N = %d", [N]));

        var dataA = [];
        for (int i = N - 1; i >= 0; i--)
            dataA[i] = N - i;
        var builtin = benchmark(function() {
            using static compare;
            dataA.sort(compare);
        });
        display_result("sort with .sort BUILTIN", builtin);

        var dataB = [];
        for (int i = N - 1; i >= 0; i--)
            dataB[i] = N - i;
        var query = benchmark(function() {
            using static compare;
            using Rosella.Query.as_queryable;
            as_queryable(dataB).sort(compare);
        });
        query.set_base_time(builtin.time());
        display_result("sort with Rosella Query", query);
    }

    function compare(var a, var b) {
        if (a > b) return 1;
        if (a == b) return 0;
        return -1;
    }

It's worth pointing out that I'm using a new prototype "Benchmark" library to
help with the benchmarking. This library is only a few days old and in flux,
but eventually it should be a great tool for this kind of stuff.

The results, as I mentioned, are interesting. I'll let the numbers speak for
themselves:

    N = 100000
    sort with .sort BUILTIN - 4.667606s - %100.000000 (%0.000000 diff)
    sort with Rosella Query - 3.187068s - %68.280568 (%-31.719432 diff)

The Rosella version is about *30% faster* than the builtin version, in both an
optimized parrot (shown) and an unoptimized one (not shown). This result
comes despite the fact that the Rosella version returns a copy of the array,
sorts the copy, which is extra overhead. The built-in version does not create
a copy and does the sort in place. Even with creating a copy first, my new
version still comes out significantly faster. The beauty of this sort routine
is that it's not a method. It can be used with any array-like type, including
all the built-in types for Parrot, and all of those see a nice performance
boost over the current implementation.

I wanted to try a technique of using a hybrid sort instead of using pure
Quicksort. For small arrays, other sorts like bubblesort and insertion sort
actually perform better, even though their complexity is theoretically higher.
The reason for this is that the loops in something like bubblesort are
invdividually more efficient than the iterations of quicksort, even if those
sorts typically require more iterations. Below a certain threshold, an
insertion sort will operate faster than a quicksort because of all the
recursions. With tuning, we can add a threshold. Below the threshold we use
a sort which is efficient at smaller list sizes, and especially small lists
which are mostly in order. Above that threshold we can use quicksort to
divide and conquer, which is where it really shines.

So for testing, I added in a quick hybrid approach to switch to insertion
sort below a certain array size. Here are the results of a comparative
benchmark similar to the one above but with more variants:

    N = 100000
    sort with .sort BUILTIN (presorted) - 4.791100s - %100.000000 (%0.000000 diff)
    sort with .sort BUILTIN (reversed)  - 4.817222s - %100.545221 (%0.545221 diff)
    sort with .sort BUILTIN (random)    - 7.011140s - %146.336751 (%46.336751 diff)
    sort with Rosella Query (reversed)  - 3.358541s - %70.099580  (%-29.900420 diff)
    sort with Rosella Query (random)    - 4.409708s - %92.039573  (%-7.960427 diff)
    qsort+insertion sort (reversed)     - 2.662546s - %55.572747  (%-44.427253 diff)
    qsort+insertion sort (random)       - 4.098206s - %85.537889  (%-14.462111 diff)

The baseline I am using in this test is the builtin quicksort on a presorted
array. I then try out the builtin sort on a reverse-sorted list and a random
one. Following, I use my Rosella qsort implementation I showed above on a
reversed list and a random list. Finally, I show an example hybrid sort on
a reversed list and a random one. The numbers are pretty clear. The hybrid
sort blows the other variants out of the water on all counts. Sorting the
random list with the hybrid sort is *14% faster than our built-in sort
implementation on a presorted array*. That's staggering, and telling. Plus,
it's only the tip of the iceburg. There are plenty of other techniques and
optimizations we could play with to find further improvements, and a
fledgling benchmarking system that we can use to compare them.

This is one of those rare occasions where the things we learned back in our
"Data Structures and Algorithms" classes can really come in handy. A sorting
routine is one of those things that runtimes are expected to provide, and the
default implementation is going to have performance ramifications on all the
software that uses it. If we can provide a better default sort for Parrot, all
the software that uses it will benefit from improved performance. Good tools
for benchmarking these results are the necessary first step, then we can start
fiddling with the implementations and finding the best one, or the best set.
