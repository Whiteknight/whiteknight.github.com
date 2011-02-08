---
layout: post
categories: [Parrot, ParrotTheory, GC]
title: GC For Newbies, Mark and Sweep
---

This is the third installment in the GC For Newbies series of posts. The
[first entry][gc_for_newbies_1] talked about memory and system `malloc`. The
[second entry][gc_for_newbies_2] talked about slab-style allocations. Today,
we're going to talk about how we account for allocated memory and how we
collect it using the *mark and sweep* algorithm.

[gc_for_newbies_1]: /2010/12/13/gc_for_newbies_memory.html
[gc_for_newbies_2]: /2011/02/04/gc_for_newbies_allocation.html

Most common GC algorithms, as I've mentioned other times before, consist of
two basic phases: mark and sweep.

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
    gc->mark_item($item);
}

{% endhighlight %}

Inside the `mark_item` subroutine we have to mark the item as "alive". This is
an extremely simple example, of course. The reality is that objects connect to
each other through references. c

{% highlight perl %}

sub mark_item {
    my $gc = shift;
    my $item = shift;
    foreach my $subitem ($item->references) {
        $gc->mark_item($subitem);
    }
    $item->mark_alive();
}

sub mark_item_alive {
    my $item = shift;
    $item->state = "alive";
}

{% endhighlight %}

Actually, this isn't even a complete example. Parrot uses a tri-color scheme
where all the items have one of three "colors": White, black, and grey. White
means the object is dead or unreferenced, Black means the object is alive, and
Grey means the object is in the middle of being marked. Objects which are free
do not have a color. Here's our mark function now:

{% highlight perl %}

sub mark_item {
    my $gc = shift;
    my $item = shift;
    $item->set_color("grey");
    foreach my $subitem ($item->references) {
        $gc->mark_item($subitem);
    }
    $item->set_color("black");
}

{% endhighlight %}

As a bit of reference to Parrot's system, the loop over '$item->references'
is all handled in `VTABLE_mark`:

{% highlight perl %}

sub mark_item {
    my $gc = shift;
    my $item = shift;
    $item->set_color("grey");
    VTABLE_mark($item);
    $item->set_color("black");
}

{% endhighlight %}

`VTABLE_mark` is basically a routine where every item passes all it's children
to the `mark_item` routine in the GC. This is how we find all references; we
expect each type to know where it keeps it's own data.

Every object in the system would have a "color" property that would keep track
of it's current status. GC then works like this:

1. Set all objects white.
2. Mark items black, starting with the root set
3. Anything that is still white at the end of the mark is dead and can be
   freed (if it isn't free already).

{% highlight perl %}

sub gc_mark_all {
    my $gc = shift;
    my @slabs = $gc->all_slabs;
    our @root;   # global, contains all root items

    for my $slab (@slabs) {
        for my $item ($slab->all_items) {
            $item->set_color("white");
        }
    }
    for my $item (@root) {
        $gc->mark_item($item);
    }
    for my $slab (@slabs) {
        for my $item ($slab->all_items) {
            if (!$item->is_free && $item->color eq "white")
                $slab->free_item($item);
        }
    }
}

{% endhighlight %}

GC is pretty simple conceptually when you look at a basic implementation like
this. However, what I've shown above is pretty naive and needlessly wasteful.
We've got many loops, including loops inside of loops, and we're iterating
over a memory pool which could conceivably contain hundreds, thousands, or
even millions of data objects.

The first and the third loop actually iterate over every single item in every
slab, an operation which is extremely wasteful to do even once. It's extremely
bad for us to do it twice on *every GC run*.

As a quick performance improvement, we can define `$black` and `$white` fields
on the slab instead of constants. Then, instead of having to manually swap all
black items to white, we swap the meanings of those two variables. This
has the exact same effect, but is a single operation but doesn't require a
loop. Here is this idea in action:

{% highlight perl %}

sub gc_mark_all {
    my $gc = shift;
    my @slabs = $gc->all_slabs;
    our @root;   # global, contains all root items
    for my $item (@root) {
        $gc->mark_item($item);
    }
    for my $slab (@slabs) {
        for my $item ($slab->all_items) {
            if (!$item->is_free && $item->color == $slab->white)
                $slab->free_item($item);
        }
    }
    my $temp = $slab->black;
    $slab->black = $slab->white;
    $slab->white = $slab->back;
}

{% endhighlight %}

Now, `$white` and `$black` are variables instead of hardcoded values, and we
can swap the meanings of those two variables instead of needing to iterate
over the entire slab, manually setting each flag from one color to the next.

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

Now we swap what white and black are, so all the black flags are now white
flags instead without having to iterate over the pool:

    +-------------------------------------------------------+
    |WWWW  WW  WWWW      WWW                                |
    +-------------------------------------------------------+

So there we are, we move from a simple naive mark and sweep algorithm, we
make a small improvement, and we've cut the complexity of our GC by 1/3.

This is the basic mark and sweep algorithm, which is essentially what Parrot
uses in its current gc. There are many other GC algorithms and modifications
on various algorithms. In the fourth installment of GC for Newbies I'll
discuss the generational algorithm, which Parrot is hoping to move to soon.
