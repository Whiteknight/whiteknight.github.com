---
layout: post
categories: [Parrot, ParrotTheory, GC]
title: GC For Newbies, Allocation
---

Several days ago I started a short series about GC and memory management for
people not familiar with it. The
[first entry was an overview of memory][gc_for_newbies_1]. Today we are going
to talk about allocating that memory for your program to use.

[gc_for_newbies_1]: /2010/12/13/gc_for_newbies_memory.html

Last time we talked about memory and the C standard library functions `malloc`
and `free`. `malloc` uses a linked list approach to keep track of individual
chunks of memory in the heap. This approach is flexible, but it's not always
efficient when the heap becomes fragmented and we need to perform long linear
searches for memory to allocate.

What we can do instead at the program level is to allocate *slabs* of memory.
We define that each slab is only going to contain objects of a single size.
For instance we can have an 8-byte object slab, a 16-byte object slab,
whatever we want. If I need an 8-byte allocation I can go directly to the
8-byte object slab and find the first one that's free. Every slot in the
8-byte slab is always 8-bytes. This means there is no fragmentation, and every
open slot is always able to satisfy the request for an allocated chunk.

We can further simplify this process by turning the free blocks into a linked
list. This is like malloc's approach, but instead of linking all chunks
together, we only link together free chunks. Now, instead of a linear search
for an open chunk, we can always find it immediately. Here's an example
diagram. In this diagram, `X` is an allocated block, empty spaces are
unallocated, and are linked together into a free list.

    Free--+
          |     +----------+
    +-----v-----|----------v--------------------------------+
    |XXXXX XXXXX XXXXXXXXXX XXX XXXXXXXXXXXXXXXXXXXXXXXXXXXX|
    +-----|-----^----------|---^----------------------------+
          +-----+          +---+

In this scheme, we can immediately find an open slot in the slab by taking the
first item off the free-item linked list. We pop the first item off the list.
When we free an item, we put it onto the list. There is a bit of time we have
to spend when we first get the slab to add all the items in it to the free
list, but after that initial cost allocations tend to be very fast.

The benefits to this slab-style allocator are many: Allocations are very fast
because we don't need to search for an open object. We can always find an open
slot immediately by looking at the free list pointer. If the free pointer is
empty we need to allocate a new slab, or resize our existing slab, which does
add some overhead, however. We also never have memory fragmentations because
all our objects are the same size. Any free spot is always exactly big enough
to hold the data we want to allocate. At the end of the program, when we want
to clean up our program memory, we can call `free` once on the entire slab.
It's as easy as can be.

In Parrot for instance, we have a PMC slab, or PMC "pool" that we allocate
PMCs from. The pool is actually a list of individual slabs which are
themselves chained together. This approach allows us to allocate new slabs
when we need more memory, without having to allocate too much up front. We
have a STRING pool too where we allocate strings from. We have a whole list of
pools for objects of various sizes. Some readers will remember me talking
about the [fixed-size allocator][fixed_size_allocator]. This is that.

[fixed_size_allocator]: /2009/08/02/fixedsize_allocations_for_parrot.html

The general idea in Perl is this:

{% highlight perl %}

sub allocate_fixed_size {
    my ($self, $size) = @_;
    my $slab = $self->slabs[$size];
    $item = $slab->free_list;
    $slab->free_list = $item->next;
    return $item;
}

sub free_fixed_size {
    my ($self, $size, $item) = @_;
    my $slab = $self->slabs[$size];
    $item->next = $slab->free_list;
    $slab->free_list = $item;
}

{% endhighlight %}

This example obviously only contains a single slab per object size, and
doesn't do anything fancy like detecting when the free list is empty and
allocating new memory. We can add in a single line of code (and, implicitly,
a new method) to handle this case:

{% highlight perl %}

sub allocate_fixed_size {
    my ($self, $size) = @_;
    my $slab = $self->slabs[$size];
    $slab->allocate_new_slab() unless defined $slab->free_list;
    $item = $slab->free_list;
    $slab->free_list = $item->next;
    return $item;
}

{% endhighlight %}

This is how Parrot's memory allocator works, in the most simple sense. It's a
very good design and is pretty speedy. It's not the absolute fastest algorithm
available, and there are some tweaks and improvements we could make for
modest gains.

Now that we've discussed memory and allocations, next time we are going to
start discussiong GC algorithms like *mark and sweep*.
