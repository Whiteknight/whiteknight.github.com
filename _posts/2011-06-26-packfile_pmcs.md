---
layout: post
categories: [Parrot, Packfiles]
title: Packfile PMCs Work
---

The embedding API work wasn't done in a vacuum. Getting a nice shiney new
embedding API in place was a nice thing to do, but it was only one component
of a larger goal: the removal of IMCC. Most of the embedding API work, such
as the cleanup and encapsulation of various bits of logic, was required to
start moving IMCC out of libparrot Core. Some of the details were just nice
things to do, and I figure that if I am going to be taking on a project like
this, I'm going to do my best to do it correctly. For some value of
"correctly".

Yesterday, I got started on the next phase of the IMCC work: Packfile PMCs.
First let me describe some of the issues we've been having, and then I'll talk
about how I'm going about fixing it.

In Parrot master right now, IMCC compiles `.pir` files to a `PackFile*`
structure. That structure is wrapped up in a generic PtrObj PMC and returned.
This is good because the PtrObj PMC is marked by the GC and so the underlying
`PackFile*` structure (and all the PMCs it contains) are marked too. That's
important. Also, we can pass this PtrObj PMC to various API routines which
know how to call `VTABLE_get_pointer` and extract the underlying `PackFile*`
structure from it. The downsides to this approach are that the PtrObj PMC is
just a dumb wrapper that provides no user-visible interface for working with
the `PackFile*` data. As far as the user is concerned the PtrObj is an opaque
data item that gets passed to things that know how to work with it. This is
suboptimal for a lot of reasons.

We have several other PMC types that work with PackFile structures too. The
Eval PMC type works something like a wrapper around PackFile but it has some
interesting behaviors: It inherits from Sub and pokes directly into the
already tangled and un-encapsulated internals of that type. Eval works like an
array of Subs, and integer-keyed indexing into Eval returns Sub PMCs from the
PackFile constants table. However, Eval doesn't provide any way to access
other constants, such as string, floating-point, or non-Sub PMC constants.
Eval also doesn't really provide any tools for working with Packfiles in a
more general sense: You can't read into an Eval PMC from an existing .pbc
file, or easily write the constants out to a file. You can't read into it from
a string of bytecode either.

We also have a series of Packfile PMCs that were created some time ago, mostly
to support the ability to create PBC files from PCT. These PackFile PMCs,
instead of providing an interface into existing `PackFile*` structures like
those generated by IMCC, provide an alternate implementation of them. The
Packfile PMC doesn't store a pointer to the `PackFile*` structure. Instead,
it stores all the data a `PackFile*` would store in PMC attributes, with each
sub-structure getting a separate PMC type to be loaded into. In essence, it's
not an interface to `PackFile*`, it's a duplicate of it.

Right now, as I mentioned, IMCC was returning a PtrObj PMC. That's lousy so I
wanted it to return something better. I don't like Eval and I want it to
disappear, so I decided to try to use the Packfile PMC instead. However, the
PackFile PMC doesn't have a set_pointer VTABLE for setting the `PackFile*`
structure coming out of IMCC. It also didn't have a method for accessing the
`:main` Sub from the packfile, or accessing Subs or other constants easily. I
tried adding some of this logic but quickly found that differences in the
design between the Packfile PMCs and the `PackFile*` structure were hard to
get around. Also, there's one other problem: inefficiency. Many of Parrot's
internal packfile operations only work on a `PackFile*` struct, not on any of
the available PMC types. So for those operations, the Packfile PMC needs to
create a temporary `PackFile*` structure for those operations, and then
destroy it again. Also, once we create a `PackFile*` structure in IMCC and
pass it to the Packfile PMC, we need to recurse through it and create PMCs to
mirror all the exact same data. That's something of a huge waste to do every
time we want to compile something.

Let's look at where we are heading, so we can understand why our current
options don't quite do what we needed and what I am hoping to do going
forward.

I've mentioned before on this blog that I want to work towards rewriting the
Parrot executable frontend in PIR, or some other Parrot language. To do that,
we need fully-functional packfile PMCs. Look at some of the command-line
arguments and capabilities we are going to need to have to support: `-o` to
output a file, the ability to load in a .pbc file, or load and compile a .pir
file, `-c` to compile .pir to .pbc, `-r` to compile, output to file, then
execute the .pbc file from disk, etc. That's quite a lot of stuff but not
entirely outside the realm of the possibility of Eval and Packfile PMCs.

However, the Parrot frontend is not the only program that might benefit from
being rewritten in a Parrot language. `pbc_merge` is another program I've
been looking at recently that would definitely benefit. That logic is quite
messy, and if we can hide enough of the messy logic behind API calls and then
expose those through PMC methods, we can rewrite much of that.
`pbc_disassemble` and `pbc_dump` programs too. Think about other programs that
we could create to open, analyze and modify packfiles, but we haven't yet
because the interface is so obtuse. `pbc_merge` alone needs a lot more stuff
than the parrot frontend needs: The ability to read in multiple packfiles and
add data from each to a single output packfile. This means we need to be able
to easily create a PMC from an existing .pbc file (which the Packfile PMC and
friends cannot really do well enough), we need to iterate over all the
contents of the resulting packfiles (which Eval cannot do), then combine
together all the data and write that output that to a file (which Eval cannot
do either).

Where do we go from here? Yesterday I started creating a new PMC type called
`PackfileView` in the `whiteknight/packfilewrapper` branch. The PackfileView
PMC acts like a thin wrapper around an existing `PackFile*` structure, with
methods which are thin wrappers around packfile subsystem API functions. It
doesn't inherit from Sub or anything else (so we avoid encapsulation-breaking
problems that Eval gets into), and uses the packfile subsystem API instead of
poking into those guts directly. PackfileView is basically read-only, although
some of the operations that happen on PackFile* necessarily modify its
internals. We can't make it a perfectly read-only interface, especially not if
we want to do anything interesting or worthwhile with it. However, we can
avoid most of the operations which explicitly modify the contents in
irrepairable or dangerous ways.

Since PackfileView is a thin wrapper around `PackFile*`, we can return it
directly from IMCC. Since it has a nice interface, we can interact with it
from PIR. So, PtrObj is out, PackfileView is in. All current code works
because PtrObj only had two inteface functions: get_pointer and set_pointer,
and PackfileView provides those the same way.

The IMCCompiler PMC replaced an old NCI PMC that served as the registered
compile for PIR. For backwards compatibility, IMCCompiler's invoke VTABLE
returns an Eval PMC. However, it also adds methods `.compile()` and
`.compile_file()` which are now returning PackfileView PMCs in the branch.
This provides a clear and seamless upgrade path for users while we deprecate
Eval PMC (and eventually, IMCCompiler.invoke).

So that's my plan for the Eval PMC and PackfileView. I hope it gets a green
light and gets merged into master eventually. I need to do a heck of a lot
of documenting and testing on the new PMC before we can start talking about
making any kind of switch anyay. However, what's the story with the existing
Packfile PMCs?

The existing Packfile PMCs are fine. They were built for the particular
purpose of allowing PCT and other compilers to build .pbc files. This is an
important and highly desirable goal, and not something we want to get in the
way of. However, this goal is relatively specialized: There's not much reason
why we would need to keep that functionality built into Parrot at all times.
Instead, I suggest that we move those PMCs out into a dynpmc library and make
it part of a large package of compiler-building tools. This is just a
suggestion, I don't have a strong enough opion on the matter and the existing
Packfile PMCs aren't really causing any harm where they are right now. Plus,
I'm anticipating the counter-argument that almost all of our users either are
HLL compiler projects, or are written in dynamic languages which support
runtime `eval` and therefore require compiler objects to be around at all
times, so most usages are going to require those PMCs. I don't have an answer
for that really and like I said this isn't a strong opinion that I'm prepared
to fight about.

Right now I'm working on the PackfileView PMC, getting it working well and
giving it enough functionality to completely replace Eval. Next step is to
start seriously cleaning up the packfile subsystem API and unifying lots of
bits of logic repeated throughout the codebase. After that, I'm going to do
some cleanups on the IMCCompiler PMC, then start the final bit of work to get
IMCC pulled out of libparrot and turned into a dynamically-loaded extension.
That last part is probably going to depend on some of the GSoC projects going
on this summer, so it certainly can't happen before the end of the season.

Let me close out this post with a snippet of code which is *working right now
in my branch*:

    .sub main :main :anon
        .param pmc args
        .local string exe_name
        .local string prog_name
        exe_name = shift args
        prog_name = shift_args

        .local pmc pir_compiler
        .local pmc packfileview
        pir_compiler = compreg 'PIR'
        packfileview = pir_compiler.compile_file(prog_name)

        .local pmc main_sub
        main_sub = packfileview.main_sub()
        push_eh _handler
        main_sub(args)
        exit 0
      _handler:
        .local pmc ex
        .get_results(ex)
        say "Unhanded exception:"
        $S0 = ex["message"]
        say $S0
        $I0 = ex["exit_code"]
        exit $I0
    .end

Add in a "`load_language 'PIR'`" call, and all might be well with the
universe.