Let's say we introduce a new PMC type, PMCHeap, which acts like a little mini
garbage collector with it's own memory pool. It also manages references to
all the objects it allocates, so they don't interfere with normal garbage
collection. We would change this:

    var f = new Foo();

...into this:

    var pmcheap = new PMCHeap();
    var f = pmcheap.create(class Foo);

Then later we could do something like this:

   pmcheap.dispose_all();

What if `f` in the second example wasn't a normal `Foo` instance, but was
instead a PMCHeapProxy object which pointed to a `Foo` instance inside the
PMCHeap? Follow along, this gets tricky.

If `pmcheap.create()` returns a normal object, that is a reference that the
GC is going to find. Depending on how PMCHeap is implemented, the GC might
either recognize `f` as being a valid PMC header and mark it, which means
that if the reference to `f` escapes, we could have that pointer outlive
`PMCHeap` and we end up with a disposed PMC being referenced from other places
which might expect it to still be viable. Worse yet, if PMCHeap actually
managed the memory allocation for it, when PMCHeap was collected the memory
for `f` would disappear leading to segfaults. Unless, of course, we add a
reference to the owner PMCHeap in `f`, then neither of them could be collected
until both of them were dead. Plus, since PMCHeap is just an allocator and
should be able to allocate objects of any type, we would have to add a
mostly-unused heap pointer to all PMC headers, which would be a huge waste.

Instead, we have a PMCHeapProxy which contains a secret reference to the
PMC header in the heap and a pointer back to the heap. All vtable accesses
on the proxy would check to see if the heap and the data pointer were still
valid and then forward on to the underlying data. Now we end up with a few
benefits:

1. When the PMCHeap disposes the data and frees memory, the PMCHeapProxy is
   still a valid PMC which intercepts accesses and throws exceptions instead
   of segfaults.
2. If the vtable forwarding is transparent enough, we should be able to
   convincingly make it appear that the proxy actually is a member of the
   underlying type, and there would be relatively little overhead.
3. The PMCHeapProxy's reference back to the PMCHeap should keep the heap alive
   for GC. The Heap won't have a pointer back to the proxies, so they would
   be allowed to die and be collected like normal.
4. The PMCHeap would be able to manage it's own memory without interference
   from the GC, since the memory would be separate and PMC pointer references
   from the heap would be encapsulated behind the PMCHeapProxy and never
   allowed to escape.

It's also possible that using custom heaps would allow us to have something
like shared memory for threaded applications. The GC doesn't really like
sharing, and for good reason. However, if you had a small sandbox of memory
of your own design and could manage all accesses yourself, shared memory
should be possible and probably even beneficial in many cases.

