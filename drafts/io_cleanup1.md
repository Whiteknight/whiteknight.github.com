---
layout: post
categories: [Parrot, IO]
title: IO Refactors
---

The IO subsystem is a lot like the garbage collector: So long as it *just
works* we can ignore its faults for quite a long time. The garbage collector
had performance and other issues for years before bacek went through and
finally rewrote it. His effort there saves the rest of us meer mortals from
having to touch the GC again for another couple years.

The IO system works reasonably well. It's got a decent set of features more
or less, it implements most of the important operations that our users have
needed in the past, it's not spectacularly slow (and disk or network operation
performance almost always outweighs any issues in the code that leads to those
things), and we haven't been getting a lot of error reports or feature
requests for it. In short, if it ain't broke, don't fix it.

A few days ago I was working on a ticket for moritz to add better integration
between our various IO vector PMCs (FileHandle, Socket, etc) and the
ByteBuffer PMC. ByteBuffer is what it's name implies: It's an array-like type
for working with individual bytes. ByteBuffer can be populated from and
exported to a STRING, and it is useful for certain types of operations that
need to operate on a sequence of bytes without having to worry about strings
and encodings and all that other nonsense. Mortiz's request was a reasonable
one so I sat down and made it happen. A few nights ago I merged that work in
to master with an "experimental" tag on it.

However while I was in the IO subsystem code making this happen something did
break. Not in the code, something broke inside my poor little head. The
snapping sound you hear is the poor camel's back under the load of that last
piece of straw. I've had enough of that system and its inside-out organization
and collection of half-ideas and botched refactors. I've had my fill of the
nonsense and am ready to start making things right.

And before anybody says to me, "hey Mr Whiteknight, you shouldn't be so mean,
somebody probably worked really hard to make this code do what it does", let
me just say two things: First, "Mr Whiteknight" is my father's handle and
Second, *I was one of the people who helped put IO where it is today*. I
don't feel particularly bad insulting myself or my own work, and my
contributions, though well-intentioned at the time, are a big part of why the
system is in the condition its in now. First, a history lesson.

When I joined Parrot, it sported an IO system based on layers. Layers were
arranged in a structure something like a vtable, and IO requests would be
fed through the layers. Each layer getting the output of the one before it
until the bottom layer actually spat the data out (or, read it in depending on
which way you were moving). This worked pretty well when you were trying to do
File IO on a file with a particular encoding, with buffering, through an
asychrony mechanism, etc. Actually I say it worked well but it was sort of
overkill: It was just too much infrastructure for the possible benefits and
despite the theory of allowing better code reuse there really weren't too many
different layering combinations that could be set up. Plus, layers start to
interdepend and violate encapsulation, then optimization starts prompting a
few "short cuts" where layers were flattened together. One of the earlier
things I did on Parrot, post-GSOC, was to remove some of the last vestiges of
the then-unused layering system from Parrot's IO.

The IO subsystem has something of a problem where it has a few masters and
has to be performance conscious. Many of our programs are still the kind that
shuffle data about (very much in the influence of Perl) and IO operation
performance mattered when your compiler is reading in HLL code and outputting
PIR code, then you're reading PIR code in and trying to compile it again.
Too much nonsense and everybody feels it.

In Parrot at the user level you can do IO in two ways: Through the IO PMCs
(FileHandle, mostly) and through opcodes (`say`, `print`, etc). The problem,
put succinctly, is this: We want to encapsulate logic for writing to files
inside the FileHandle PMC, but we don't want to add new IO-specific VTABLES
and we don't want to incurr the costs of method calls on every single IO
request. In other words, we didn't want the `print` opcode to just be a
thin wrapper around the `print` method on FileHandle. Such a thing, especially
if implemented naively, would have killed performance by creating nested
runloops and a whole host of other problems.

So the FileHandle holds the state, the OS-level file descriptor, the buffer
and buffering logic, flags about whether the file is open or not and whether
it's for writing or reading or appending or whatever.

The way the system is set up is that both `FileHandle.print()` and the `print`
opcode are both thin wrappers around the real routine `Parrot_io_putps`, which
does all the hard work. And, more importantly, that routine is expected to
act transparently (like the `print` opcode does) on any IO PMC type like
Socket or StringHandle. The only real way to do this, if you can't call a
method on the FileHandle and Socket PMC is to use a large switch-statement:

    switch (handle->vtable->base_type) {
        case enum_class_FileHandle:
            ...
        case enum_class_Socket:
            ...
        case enum_class_StringHandle:
            ...
        default:
            Parrot_pcc_invoke_method_from_c_args(..., handle, "print", ...);
    }

I've obviously glossed over all the details, but this is the general form
of that routine. You'll notice several things.

1. If we want to add a new IO type to Parrot core we need to add a new entry
   to the switch statement in *every IO API routine that needs to care about
   PMC type*.
2. If the user passes in an Object, something defined at the PIR level, we
   do fall back to calling the method, because we can't do anything else
   intelligently.
3. We can't really subclass FileHandle or Socket from the user level, because
   it would fail the `base_type` test, and wouldn't be able to handle the
   low-level structure accesses from that point forward anyway.

Point number 2 is particularly interesting because the `FileHandle.print()`
method calls `Parrot_io_putps`, which may turn around and call the `.print()`
method. I don't think there's any real chance for endless recursion here, but
it's clearly an example of poorly separated concerns and poor encapsulation.
Either the method should call the IO API or the IO API should call the method
but we can't be doing both.

There are a few other issues worth mentioning, which I'll just dump rapid-fire
without much explanation:

* We don't have a separate Pipe type. Instead, FileHandle can be opened in
  "pipe mode" to write to a separate process or read output from a separate
  process.
* We have limited buffering, but only on FileHandle and we cannot configure
  buffers for input and output separately.
* We don't really have encodings set up in any consistent way, so it's very
  possible, though I haven't worked out all the details, to write strings with
  different encodings to a file. This is especially true if we're using
  buffers
* FileHandle logic is considered to be the default and is given deference in
  the code. Pipe logic is unified with file logic at a very low level. Socket
  and StringHandle are treated as bolted-on spare parts and don't benefit from
  hardly any code sharing or unified architecture.
* Several functions in the IO subsystem are poorly or inconsistently named and
  implemented.

So that's the system we've got. What do I want to do to fix these issues?

The first thing I've suggested is to break up IO functionality into an
`io_vtable` of function pointers, similar to how the `STR_VTABLE`, the
sprintf dispatch mechanism, the packfile segment dispatch table and other
similar mechanisms in Parrot work. Instead of the PMC `print` method calling
`Parrot_io_puts`, instead it would call `io_vtable->print_s()`.
`Parrot_io_puts` would make exactly the same call, and each IO type would have
a separate type-specific implementation somewhere.

The second thing I want to do is break out buffering so that instead of being
a detail of the FileHandle PMC a buffer is a separate struct which can be
attached to any IO type as desired. And, even better, we can attach multiple
buffers to an IO stream, at least one each for input and output, configured
separately. The buffering API, which will be cleaned up and properly
encapsulated, will take a pointer to the `io_vtable` routine being invoked and
will pass data through transparently as required. A thin wrapper PMC type,
IOBuffer, would allow references to buffers to be accessed and configured
directly, which would be very useful in some cases.

Imagine, if I may go off on a short tangent, a threaded system where one
worker task had a reference to a buffer and continuously made sure it was
filled in the background while another worker task read bits and pieces from
the buffer very quickly. It would be possible, through careful choice of
algorithm, to do such a thing lock-free. Feel free to replace "file" with
"socket" or "pipe" in the example above too. Imagine also a system where we
can transparently use `mmap` (or it's windows equivalent) to map a file
to memory as part of the buffer, and keep working with it that way.

The third thing I want to do is start teasing apart the logic for Pipes from
the file logic. I'll create a separate `io_vtable` for pipe operations, and
use that inside FileHandle when we're in pipe mode. Eventually we'll be able
to create a separate type, divide out all the logic completely, and get to
work on really interesting stuff like feasible 2-way and 3-way pipes.

The fourth thing I want to do is start setting up interfaces so that IO
operations including buffering, low-level IO, file descriptor manipulation
and other things become more accessible at the PIR level so users can make
better use of these tools.

I've started sketching out many of these ideas in the
`whiteknight/io_cleanup1` branch. cotto seems to agree with the general
direction and I haven't heard any complaints so far, so I'm prepared to move
forwardwith the idea soon. Maybe start shuffling code as early as this
weekend.




