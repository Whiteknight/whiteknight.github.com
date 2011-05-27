A few days ago I was reminded of an old ticket about sorting. We have 10
basic array types in Parrot:

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
isn't implemented already.

However, I feel like that's not a great solution. We shouldn't have to copy
and paste sort method code into all these types. We should either inherit the
code among all of these, or write up a sorting routine somewhere else that can
work on all array-like types. For a variety of reasons I don't think
inheritance is the way to go on this occasion because there isn't enough
common functionality between these array types that we could inherit easily.
Plus, I had a hunch that the performance was not so great on our current
quicksort implementation because it was written in C and used nested runloops
to execute the comparion routines. As my recent benchmarks showed, nested
runloops are performance killers.

So I implemented a new sort routine as part of the new Rosella Query library.
The initial implementation was naive, but with a few easy optimizations I
up with some *interesting* results. Here is my current quicksort
implementation in winxed:

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

Here is the benchmark code:

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

    function display_result(string name, var result) {
        print(sprintf("%s - %s", [name, result]);
    }


It's worth pointing out that I'm using a new prototype "Benchmark" library to
help with the benchmarking. This library is only a few days old and in flux,
but eventually it should be a great tool for this kind of stuff.

The results, as I mentioned, are interesting. I'll let the numbers speak for
themselves:

    N = 100000
    sort with .sort BUILTIN - 4.667606s - %100.000000 (%0.000000 diff)
    sort with Rosella Query - 3.187068s - %68.280568 (%-31.719432 diff)

The Rosella version is about 30% faster than the builtin version, in an
optimized Parrot. The percent differences are about the same for an
unoptimized Parrot, although the total times are higher for both. The beauty
of this sort routine is that it's not a method. It can be used with any
array-like type, including all the built-in types for Parrot, and all of those
see a nice performance boost over the current implementation.

I'm happy with this for now, but eventually I would like to experiment with
some alternate implementations for even better performance. An Introsort
variant would have similar performance while avoiding worst-case performance
on certain input sequences. Implementing something like bubblesort for short
sub-arrays to prevent recursion might also have some performance boosts,
though that would require some tuning and experimenting to test it out. Lucian
has also clued me in to Timsort, an interesting mergesort/insertionsort
algorithm used by Python that seems to have pretty good performance for
input arrays that are mostly sorted already. I'm not going to put too much
effort into this now because I have a working sort that is already faster than
what Parrot has been providing, but I'l get to it eventually if nobody else
wants to play with it first.

