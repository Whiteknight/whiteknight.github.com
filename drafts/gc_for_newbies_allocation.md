What we can do instead at the program level is to allocate *slabs* of memory.
We define that each slab is only going to contain objects of a single size.
For instance we can have an 8-byte object slab, a 16-byte object slab,
whatever we want. If I need an 8-byte allocation I can go directly to the
8-byte object slab and find the first one that's free.

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
because we don't need to search for an open object. We can always find an open
slot immediately by looking at the free list pointer. We also never have
memory fragmentations because all our objects are the same size. Any free spot
is always exactly big enough to hold the data we want to allocate. At the end
of the program, when we want to clean up our program memory, we can call
`free` once on the entire slab. It's as easy as can be.

In Parrot for instance, we have a PMC slab, or PMC "pool" that we allocate
PMCs from. We have a STRING pool where we allocate strings from. We have a
whole list of pools for objects of various sizes.

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
