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
