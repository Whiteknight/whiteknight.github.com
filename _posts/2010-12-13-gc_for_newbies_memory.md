A few people have requested it recently, so this post is going to be a
basic introduction to the concepts of GC and memory management in general. I'm
not going to get into too many of the nitty-gritty details, and I'm going to
try to give short code examples in higher-level languages like Perl and C# so
everybody can follow along.

In the C standard library we have two functions for managing dynamic memory:
`malloc` to allocate a block of memory and `free` to return that memory to the
system. Blocks of memory returned by `malloc` are allocated on the system
*heap*, as opposed to program flow control and local variables which are
allocated on the system *stack*. Stack space is pretty quick and easy to use
but it's limited in size and has some pretty strict rules about when and how
it is used. For instance, stack space is typically specified at *compile
time*. That is, the layout of the stack is determined from the structure of
your source code. I can take a look at any arbitrary C source code and tell
you what your stack allocation will probably look like (ignoring the massive
mangling of an optimizing compiler, which I won't talk about here). Space that
you allocate on the stack for local variables ands things will be
automatically allocated and managed by the compiler, and will be automatically
cleaned up at the end of every function. I'm dramatically oversimplifying
here, but the fact is this: Stack space is rigid and is determined at compile
time.

In C, you can allocate an integer on the stack with the simple declaration:

{% highlight c %}

int main (void) {
    int x;   /* Allocated on the stack */
    ...

{% endhighlight %}

Heap space, on the other hand, is a lot more dynamic and flexible, but has to
be accessed through `malloc` and needs to be explicitly freed with `free`.
It's more work for the programmer, but it's flexible enough to allocate what
we want when we want it.

If you allocate a bunch of memory with `malloc` and never `free` it, you have
what's called a *memory leak*. If you make many allocations without freeing
them, you can exhaust the available memory of your computer. This is unlikely
because many modern computers have huge amounts of memory available through
the use of virtual memory pages.

In short, think about memory this way: The storage your computer has is your
hard disk. Hard disks are big but slow. When you turn the computer on, it can
start to read certain bits of data from the slow disk into RAM. RAM acts like
a smaller, but faster data cache, so things that your processor wants to work
on can be made more readily available than if they were on disk. The more you
work with RAM, the faster your program; the more you work with the hard disk,
the slower your program. If your computer has 1Gb of RAM and your program
allocates 2Gb of memory through `malloc`, some of that data is going to have
to be stored on slow disk instead of in fast RAM. This creates a phenominon
called "Thrashing".

Thrashing happens when we have more memory allocated than we have space in
RAM to hold. Some of the data gets stored on the hard disk. When we need to
access data from the disk, we need to first clear out space in RAM, by writing
old data to disk. Then when we have everything saved we read the data we need
from disk into RAM. Constantly shuffling data between disk and RAM is slow
because disk is slow.

`malloc` is a general-purpose utility that everybody uses to allocate memory
blocks of all sorts of different sizes. Internally, malloc typically uses
a linked-list to keep track of many blocks of multiple sizes. We start with a
big empty block:

    +-------------------------------------------------------+
    |sn                                                     |
    +-------------------------------------------------------+

In the diagram above, `s` is the size of the block, and `n` is a pointer to
the next block. In this case, `n` is a NULL pointer and `s` is 54. Somebody
calls `malloc(5)`, and now our heap looks like this:

    +-------------------------------------------------------+
    |57     sn                                              |
    +-------------------------------------------------------+

Now the first block has a size of 5 and a next pointer to location 7 (5 slots)
for the requested memory and 2 slots for the `s`/`n` values. Now, somebody
calls `malloc(3)` and we end up with this:

    +-------------------------------------------------------+
    |57     3B   sn                                         |
    +-------------------------------------------------------+

...and so on and so forth. Now, somebody does a `free()` on the first block
and then immediately calls `malloc(2)`. This creates this weird configuration:

    +-------------------------------------------------------+
    |24  17 3B   sn                                         |
    +-------------------------------------------------------+

Now we have an allocated block of 2, an unallocated block of 1, and an
an allocated block of 3 all in a row. As we continue to `malloc` and `free`
over and over again with various sizes, memory starts to get *fragmented*.
Every time I call `malloc`, it needs to iterate over all blocks in the pool
looking for a free block that's large enough to satisfy the request. The more
I allocate and the more fragmented memory becomes, the worse the performance
of this algorithm is.

I briefly mentioned the idea of "virtual memory" in passing above. It's time
to discuss that a little bit more here. In most modern systems, memory is
broken up into chunks called "Pages". A page is fixed size, such as 4096 bytes
or similiar. These pages are really the smallest size of memory that the
operating system manages.

In your system you have "main memory" (RAM) and "extended memory" (disk). The
system can use extended memory to make it look like we have more main memory
than we really do. This magic trick is called "virtual memory".

Virtual memory works by having Pages and a Page Lookup Table. The Page Lookup
Table keeps track of where a page is in virtual memory. A page could be on
the disk or in RAM. It can move around in RAM too. The processor uses the
lookup table to find the page and update memory address values from the
address where the page is actually located to the address that the application
thinks it is at. The details of the algorithm are a bit messy, but the
important part is this: Pages are the basic building block of memory, and
Pages can move between RAM and disk depending on use.

Infrequently-used pages can be saved to disk to make more space in RAM for
data that is used frequently. When we try to access a page which is not
currently in RAM, that triggers a *page fault* where the processor has to
stop what it is doing and load the missing page back into RAM for use.
Repeatedly triggering page faults is thrashing.

When we use `malloc`, we can typically avoid problems of fragmentation if we
allocate an entire Page at once, or a mulitiple of the page size for very
large objects.

So there is a basic overview of memory, the difference between stack and heap,
the role of disk and RAM, and a general overview of virtual memory. In
coming posts I will talk about making more efficient allocations, and then
talk about GC and the various algorithms that I have discussed on this blog.
