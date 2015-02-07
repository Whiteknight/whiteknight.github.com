---
layout: post
categories: [Parrot, Packfiles, IMCC]
title: New IMCC Tag Syntax
---

I've talked on a few occasions about some of the ugliness in IMCC and the
packfiles subsystem. Yesterday I merged in my newest cleanup effort, a branch
called `whiteknight/pbc_pbc`. That branch removed some dead code and added
some new details to support some ongoing deprecations. It added, among other
things, a new variant of the `load_bytecode` op that has semantics much
closer to what we want them to be in the future. Here are some examples:

    load_bytecode "foo.pbc"       # old style
    $P0 = load_bytecode "foo.pbc" # new style

The first variant is magical and does a lot of work. It can take a filename
argument which is either a .pbc, a .pir, or a .pasm file. It does some really
ugly extension detection, and will compile the file if necessary. It loads it
directly otherwise. Once it compiles or loads the file, it uses the horrible
and deprecated `do_sub_pragmas` function to loop over all subs and execute the
ones marked `:load`. Finally, it adds the name of the file (without the file
extension) to a cache so we don't try to load the same file again.

The second variant is much simpler. It only takes .pbc files, it loads them,
and it returns the PackfileView for it. Then the user can do anything she
wants with respect to finding and executing `:load` or `:init` functions or
anything else. This version does not currentls have a cache to prevent
multiple loads of the same file, but we're working to come up with a good way
to add it. The new opcode will have cache behavior, one way or another, by the
end of next week.

Today I started yet another branch, whiteknight/imcc_tag to start taking this
idea to the logical next level: User-defined function pragmas. Here's an
example of a code file ("test.pir") that a user could create:

    .sub 'Foo' :tag("load", "init")
        say "Foo!"
    .end

    .sub 'Bar' :tag("init", "something-else")
        say 'Bar!'
    .end

Notice the new `:tag` syntax, instead of the old `:load` and `:init` flags.
Here is some code that makes use of it:

    $P0 = load_bytecode "test.pbc"
    $P1 = $P0.'subs_by_flag'("something-else")
    $P2 = $P1[0]
    $P2()           # "Bar!"

It should be pretty clear: This is much more flexible and usable than the
current system. Even better than that, this code *works today* in my branch.
The changes to IMCC syntax happened yesterday and were much easier than I
expected. I filled in most of the structural details this morning, and those
also went pretty quickly. I tracked down a few bugs and did some quick ad hoc
testing before pushing my changes for the world to see.

The way I implemented tags was through a list of index pairs. The first index
in the pair is an index into the PMC constants table. The second is an index
into the STRING constants table. Any PMC therefore can be mapped to any
string in the table, and automatic deduplication of string constants means the
lookup for all Subs with a given tag is very fast: a tight loop over an array
of integers. For most cases, I suspect it's faster than the old-style flag
lookups, although I haven't done any benchmarks yet. Avoiding a loop over all
pmc constants and calling `VTABLE_isa` on each to see if it's a `"sub"` or not
and then checking the Sub flags should produce big savings. Of couse,
`load_bytecode` operations are relatively uncommon so it won't add up to
anything too substantial for most programs in terms of saved wall-clock time.

I've got a lot of work to do in this branch still. I need to update the
various packfile PMCs to be able to read and write these new tags, and I need
to test the crap out of it. I don't know when I'll be ready to merge, but I'm
hoping to have it in before 3.7 so I can get in the deprecation notice for
`:load` and `:init` as early as possible.

Eventually, I would like to be able to replace most of the built-in Sub
pragma flags with custom strings. This will lead to a huge cleanup of ugly
bit-twiddling code, some speedups (again, they will be modest), and lots of
cool new flexibility.

In personal news, we're finally buying a new house, and are making settlement
this thursday. This week will probably be taken up by packing, moving,
paperwork, and other pleasantries of the process. Real hard-core hacking
probably won't happen much for the next two weeks at least.

