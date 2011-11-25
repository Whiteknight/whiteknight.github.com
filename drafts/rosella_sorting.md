A while back I was playing around with some [sorting algorithms and benchmarks]
[sorting_benchmarks] for Parrot. I had a quicksort hybrid implementation written
in winxed that was consistently outperforming the built-in C implementation of
quicksort by about 20%. I decided that I wanted to play with a few more
algorithms, especially algorithms which were known to have different performance
characteristics on different input types.

For GCI I created two new tasks asking for alternate sort implementations. The
first was a [Timsort][] implementation, and the second was for [Smoothsort][].
GCI students Yuki'N and blaise each delivered a winxed implementation of their
respective sort, and now I'm able to do some interesting benchmarks showing how
they work on different inputs. Here are some of those results:

    N = 100000
    SORT_TRANSITION = 6

    FORWARD-SORTED (PRESORTED) BENCHMARKS

    sort with .sort BUILTIN (reversed)
        9.872862s - %100.000000
        Number of items out of order: 0
    sort with Rosella Query (reversed)
        8.623591s - %87.346416 (-%12.653584 compared to base)
        Number of items out of order: 0
    qsort+insertion sort (reversed)
        7.812607s - %79.132142 (-%20.867858 compared to base)
        Number of items out of order: 0
    timsort (reversed)
        0.693221s - %7.021481 (-%92.978519 compared to base)
        Number of items out of order: 0
    Smoothsort (reversed)
        2.154514s - %21.822589 (-%78.177411 compared to base)
        Number of items out of order: 0

    REVERSE-SORTED BENCHMARKS

    sort with .sort BUILTIN (reversed)
        10.000436s - %100.000000
        Number of items out of order: 0
    sort with Rosella Query (reversed)
        8.891811s - %88.914232 (-%11.085768 compared to base)
        Number of items out of order: 0
    qsort+insertion sort (reversed)
        8.555817s - %85.554438 (-%14.445562 compared to base)
        Number of items out of order: 0
    timsort (reversed)
        0.812737s - %8.127015 (-%91.872985 compared to base)
        Number of items out of order: 0
    Smoothsort (reversed)
        10.521473s - %105.210141 (+%5.210141 compared to base)
        Number of items out of order: 0

    RANDOM BENCHMARKS

    sort with .sort BUILTIN (random)
        13.536509s - %100.000000
        Number of items out of order: 0
    sort with Rosella Query (random)
        12.566452s - %92.833773 (-%7.166227 compared to base)
        Number of items out of order: 0
    qsort+insertion sort (random)
        11.859363s - %87.610203 (-%12.389797 compared to base)
        Number of items out of order: 0
    Timsort (random)
        14.461384s - %106.832449 (+%6.832449 compared to base)
        Number of items out of order: 0
    Smoothsort (random)
        12.764498s - %94.296823 (-%5.703177 compared to base)
        Number of items out of order: 0

These benchmarks show some things we already knew: the built-in Quicksort
implementation from Parrot is poor across the board. The Quicksort variant
that's in Rosella is better, and my hybrid quicksort+insertion sort variant is
better still. What's interesting to see is how Timsort and Smoothsort perform
on these workloads.

Timsort is designed to work well with "real-world" data which is already sorted
or already partially sorted. It identifies runs in the data that are already
mostly sorted, and merges subsequent runs together. Timsort also has the nice
feature of identifying runs which are already reverse-sorted and does a very
fast reverse to get them ready for merging. We see that the Timsort blows all
other challengers out of the water when the array is already sorted and already
reverse-sorted. In these instances, the analysis stages of Timsort figure out
that no sorting is ever necessary.

Smoothsort constructs a special type of heap from the input data, and uses basic
balancing operations on the heap to find the largest value, extracts it, and
rebalances the heap. It works very well for the pre-sorted case, but not quite
as well as Timsort because it does need to construct this heap first, then
iterate over it. Smoothsort goes so quick because the heap rebalancing
operations when the array is already sorted are essentially free. So Smoothsort
is quick on a pre-sorted array, but we also see that it's terrible when the
input array is reverse sorted. Timsort still does very well in this case.

When the array is completely random, the story is a little bit different. Both
Timsort and Smoothsort lose to the quicksort implementations for completely
random data. Timsort actually is *worse* than Parrot's built-in quicksort, which
is weird. Smoothsort is in the same ballpark as the Rosella quicksort, but
is a few percent off of the hybrid sort.

If I had to put together a small report-card for these algorithms under all
conditions, it would look something like this:

    algorithm       pre-sorted      reversed        random
    ------------------------------------------------------
    quicksort       B               B               A
    hybrid sort     B+              B+              A+
    Timsort         A+              A+              C
    Smoothsort      A               C               B+

At a glance you can really see where each algorithm excels.

It's worth nothing here that all these implementations are relatively naive and
unoptimized. So we can say that "But Algorithm X could be optimized to be even
better!", but the same can be said about all of them. I'll be doing some of that
in the coming days, but I don't expect any radical changes.

What I would like to do in the future is provide a default sort implementation
but also have the sorting interface take some sort of optional "hint" flag that
can tell the sorter about certain proper

### Addenum about Big-O and Algorithms

Everybody will tell you that quicksort is `O(n log n)` on average, and has a
pathological worst-case that's `O(n^2)`. People will also happily point out
that something like Timsort has a best case of `O(n)`. What these simple
expressions ignore are all the details. The pathological worst case of Quicksort
requires a very specific input ordering and an absolute worst selection of the
pivot element at each recursion. Even basic modifications to the algorithm or
using a hybrid approach completely eliminates these worst-cases. Without such
basic modifications the worst-case is certainly possible, but relatively
unlikely.

What people also forget when talking about big-O notation are the coefficients.
When I say that quicksort has average complexity of `O(n log n)`, what I really
mean is that the amount of time it takes is:

    t = c * n * log(n) + f(n) + d

Where `f(n)` is any function that grows more slowly than `n * log(n)`, and
`c` and `d` are arbitrary coefficients. The quicksort algorithm, properly
implemented and optimized, has very low coefficients. The algorithm requires
very little setup (`d`) and performs relatively few operations per iteration
(`c`). The reason why Parrot's built-in quicksort performs so poorly is because
the `c` there involves recursive PCC calls and nested runloops, so `c` is
unnecessarily large.  Just by having it all run in a single runloop we can
drop `c` enough to beat the original implementation. The two implementations use
almost exactly the same algorithm, so it's differences in `c` (and, to a smaller
extent, differences in `d`) that result in the timing improvements.

Insertion sort, and this is why I picked it to be part of my hybrid quicksort,
has `O(n^2)` complexity, but with very low `c`. Below a certain threshold, the
quick sort algorithm becomes dominated by recursion calls and stack management,
and below that threshold the insertion sort performs better. Basically, there's
a very narrow window below which insertion sort's `O(n^2)` is lower than
quicksort's `O(n log n)`. By switching algorithms below that threshold, we can
squeeze out a few extra percentage points in performance savings. I could easily
have used something else like Bubblesort here for the same kind of effect.

Timsort, because it does involve ahead-of-time analysis steps to detect
pre-sorted runs is always going to be at something of a disadvantage when faced
with a purely-random input. Assuming it's core sorting algorithm is as efficient
on random data as quicksort is (it isn't, but we can pretend), Timsort is always
going to lose those benchmarks because quicksort will be just as fast during the
sorting and wont have a forward analysis phase.

Smoothsort is very interesting from a mathematical perspective, and it doesn't
do ahead-of-time analysis like Timsort does. Of course, it does need to
construct that special heap, which likewise acts like a damping agent on
overall performance results. Smoothsort does very well on pre-sorted data,
reasonably well on random data, and completely falls apart when the data is
almost exactly reverse-sorted. I suspect we could do some kind of analysis there
to detect the worst case and build our heaps backwards, but that would further
cut into the performance cost of the common cases.
