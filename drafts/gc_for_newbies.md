A few people have requested it recently, so this post is going to be a
basic introduction to the concepts of GC and memory management in general. I'm
not going to get into too many of the nitty-gritty details, and I'm going to
try to give short code examples in higher-level languages like Perl and C# so
everybody can follow along.

In the C standard library we have two functions for managing memory: `malloc`
to allocate a block of memory and `free` to return that memory to the system.
Blocks of memory returned by `malloc` are allocated on the system *heap*, as
opposed to program flow control and local variables which are allocated on
the system *stack*. Stack space is pretty quick and easy to use but it's
limited in size and has some pretty strict rules about when and how it is
used. For instance, stack space is typically specified at *compile time*. That
is, the layout of the stack is determined from the structure of your source
code. I can take a look at any arbitrary C source code and tell you what
your stack allocation will probably look like (ignoring the massive mangling
of an optimizing compiler, which I won't talk about here). Space that you
allocate on the stack for local variables ands things will be automatically
allocated and managed by the compiler, and will be automatically cleaned up
at the end of every function. I'm dramatically oversimplifying here, but the
fact is this: Stack space is rigid and is determined at compile time.

In C, you can allocate an integer on the stack with the simple declaration:

{% highlight c %}

int x;   /* Allocated on the stack */

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

In short, think about it this way: The storage your computer has is your hard
disk. Hard disks are big but slow. When you turn the computer on, it can start
to cache certain bits of data in RAM for faster access. RAM is essentially a
cache for your hard disk. The more you work with RAM, the faster your program.
The more you work with the hard disk, the slower your program. If your
computer has 1Gb of RAM and your program allocates 2Gb of memory through
`malloc`, some of that data is going to have to be stored on slow disk instead
of in fast RAM. This means that the operating system is going to have to be
accessing the data you need on the slow disk, and shuffling data back and
forth between disk and RAM. This is called *thrashing* and is an absolute
performance killer.

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
I allocate, and the more fragmented memory becomes, the worse the performance
of this algorithm is.

What we can do instead is allocate *slabs* of memory. We define that each
slab is only going to contain objects of a single size. For instance we can
have an 8-byte object slab, a 16-byte object slab, whatever we want. If I need
an 8-byte allocation I can go directly to the 8-byte object slab and find
the first one that's free.

We can further simplify this process by turning the free blocks into a linked
list. Here's an example diagram. In this diagram, `X` is an allocated block,
empty spaces are unallocated.

    Free--+
          |     +----------+
    +-----|-----|----------|--------------------------------+
    |XXXXX XXXXX XXXXXXXXXX XXX XXXXXXXXXXXXXXXXXXXXXXXXXXXX|
    +-----|-----|----------|---|----------------------------+
          +-----+          +---+

In this scheme, we can immediately find an open slot in the slab by taking the
first item off the free-item linked list. We pop the first item off the list.
When we free an item, we put it onto the list. There is a bit of time we have
to spend when we first get the slab to add all the items in it to the free
list, but after that initialization it is very fast.

The benefits to this slab-style allocator are many: Allocations are very fast
beause we don't need to search for an open object. We can always find an open
slot immediately by looking at the free list pointer. We also never have
memory fragmentations because all our objects are the same size. At the end
of the program, when we want to clean up our program memory, we can call
`free` once on the entire slab. It's as easy as can be.

In Parrot for instance, we have a PMC slab, or PMC "pool" that we allocate
PMCs from. We have a STRING pool where we allocate strings from. We have a
whole list of pools for objects of various sizes.

Now that we've covered memory and allocations, it's time to talk about GC.
GC, as I've mentioned other times before, consists of two basic phases: mark
and sweep.

In the mark phase, we account for all the live memory. In the sweep phase we
free anything that isn't alive. Here is our familiar slab, where `X` are
allocated objects and spaces are unallocated.

    +-------------------------------------------------------+
    |XXXXX XXXXX XXXXXXXXXX XXX XXXXXXXXXXXXXXXXXXXXXXXXXXXX|
    +-------------------------------------------------------+

In the mark phase, we start with a root set. In the case of Parrot, the root
set is the interpreter itself.

{% highlight perl %}

foreach my $item (@root) {
    mark_item($item);
}

{% endhighlight %}

Inside the mark_item subroutine we have to mark the item as "alive". This is
an extremely simple example, of course. Objects connect to each other
through references, so we have to recurse:

{% highlight perl %}

sub mark_item {
    my $item = shift;
    foreach my $subitem ($item->references()) {
        mark_item($subitem);
    }
    mark_item_alive($item);
}

{% endhighlight %}

Actually, this isn't even a complete example. Parrot uses a tri-color scheme
where all the items have one of three "colors": White, black, and grey. White
means the object is dead, Black means the object is alive, and Grey means the
object is in the middle of being marked. Objects which are free do not have
a color. Here's our mark function now:

{% highlight perl %}

sub mark_item {
    my $item = shift;
    mark_item_grey($item);
    foreach my $subitem ($item->references()) {
        mark_item($subitem);
    }
    mark_item_black($item);
}

{% endhighlight %}

Every object in the system would have a "color" property that would keep track
of it's current status.

GC then works like this:
1. Set all objects white.
2. Mark items black, starting with the root set
3. Anything that is still white at the end of the mark is dead and can be
   freed (if it isn't free already).

{% highlight perl %}

for my $item (@slab) {
    mark_item_white($item);
}
for my $item (@root) {
    mark_item($item);
}
for my $item (@slab) {
    if (!is_free($item) && is_white($item))
        free_item($item);
}

{% endhighlight %}

It's actually very simple when you look at a basic naive implementation like
this. This is actually extremely naive, there's no reason why we need three
loops. The first and the third loop actually iterate over every single item
in the slab which is extremely wasteful to do even once, but it's a huge
pain to do it twice.

As a quick performance improvement, we can define `$black` and `$white` as
variables instead of constants. Now:


{% highlight perl %}

for my $item (@root) {
    mark_item($item);
}
for my $item (@slab) {
    if (!is_free($item) && is_white($item))
        free_item($item);
}
swap(\$black, \$white);

{% endhighlight %}

Here's an example. We start off with the same memory diagram. W is white,
B is black, and space is free:

    +-------------------------------------------------------+
    |WWWWWWWWWWWWWWWWWWWWWWW                                |
    +-------------------------------------------------------+

Now, we go through and mark items Black, starting with the root:

    +-------------------------------------------------------+
    |BBBBWWBBWWBBBBWWWWWWBBB                                |
    +-------------------------------------------------------+

Everything white is now free:

    +-------------------------------------------------------+
    |BBBB  BB  BBBB      BBB                                |
    +-------------------------------------------------------+

Now, we `swap(\$black, \$white)`, so all the black flags are now white flags
instead without having to iterate over the pool:

    +-------------------------------------------------------+
    |WWWW  WW  WWWW      WWW                                |
    +-------------------------------------------------------+

So there we are, we move from a simple naive mark and sweep algorithm, we
make a small improvement, and we've cut the complexity of our GC by 1/3.


