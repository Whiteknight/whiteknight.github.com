I've been doing a lot of work recently on various new features, various
deprecations, and various other changes in Parrot. The goal of all these
these things is clear: A more sane, more understandable, cleaner, more
performant Parrot. Whether the path I'm pursuing is going to achisve all these
things in the long run or whether they are the best method of achieving them
is certainly open to debate, but I think it's a pretty good plan and I'm
putting my energies behind it. But, what is the plan, exactly?

It turns out that the plan is pretty big, and has a few more steps than I
would like. This blog post is as much an attempt for me to explain myself to
the readers as it is an attempt for me to commit the various ideas in my head
down someplace safe for reference. If I've learned anything in my life so far,
it's that if I don't write a thought down, it does not and never has existed.

Last weekend I added `:tag()` syntax to IMCC. Now, instead of writing this:

    .sub 'foo' :load
        ...
    .end

You would instead write something like this:

    .sub 'foo' :tag("load")
        ...
    .end

And when you want to get a library and initialize it, you used to do something
like this:

    load_bytecode "foo.pbc"

But now you're going to be doing something like this:

    $P0 = load_bytecode "foo.pbc"
    $P1 = $P0.'subs_by_flag'("load")
    $P2 = iter $P1
  loop_top:
    unless $P2 goto loop_bottom
    $P3 = shift $P2
    $P3()
    goto loop_top
  loop_bottom:

I know this all looks like extra PIR code to write, but I have two tricks up
my sleeve: The first is that this new approach is *faster* than the old one,
even if it looks like more code to write and therefore to be executed. The
second is that people don't, and shouldn't, be writting PIR by hand
anymore. It's trivial for a compiler like Winxed or NQP to generate this code
automatically, without the user having to do anything special. Libraries like
Rosella are also going to be providing tools to manage, cache, and initialize
libraries. It's only a few lines of code, so any other library, framework, or
compiler can implement it too, if you don't want to rely on Rosella.

The big benefit here is flexibility. Let's say that the user has a library
which absolutely requires 2-step initialization. One set of functions must be
executed first (such as functions which register class definitions and create
space for global variables) and one set of functions must be executed second.
Now, we can simply give them two different tags:

    .sub 'first' :tag("load-phase1")
        ...
    .end

    .sub 'second' :tag("load-phase2")
        ...
    .end

And when you load the library, you can do something like this:

    $P0 = load_library "foo.pbc"
    $P1 = $P0.'subs_by_flag'("load-phase1")
    'execute_all_subs'($P1)
    $P1 = $P0.'subs_by_flag'("load-phase2")
    'execute_all_subs'($P1)

OR, let's say you have a language like Perl6, which offers `BEGIN` and `END`
blocks, among others. Currently PIR doesn't have any flags whatsoever to be
used for specifying which subs are to be executed at the end of the program
run for cleanup. With the new system, this is trivially easy to do. Currently
PIR doesn't have a way to mark with metadata any arbitrary group of subs, only
the `:load` and `:init` ones. In the new system, you do.

We run into a few problems, however. First is that the load `load_bytecode_s`
opcode automatically executes the `:load` functions while the new
`load_bytecode_p_s` variant does not. Also, the two are using different
caches, so we cannot currently intermingle them. The old `load_bytecode_s`
does the old `do_sub_pragma` song and dance: Every time you `load_bytecode_s`,
it loops over all constants in the constant table, checks if each is a Sub,
checks if each has the `:load` flag, executes it if so (in a new, expensive,
nested runloop), and then *clears the flag*. That way, if you re-load the same
packfile somehow, it will attempt to re-initialize but all the flags would
have been cleared. Nevermind the fact that we really need packfiles and their
contents to remain constant at runtime for a variety of performance and
concurrency reasons, and never mind the fact that the real solution is to
not automatically intermingle library initialization with other unrelated
operations like program execution, packfile loading, or even compilation.
Part of the problem also is that the old `load_bytecode_s` mechanism loads a
new PackFile from the file and *merges it into the single global packfile*,
again violating our need for immutable packfiles and adding all sorts of
unnecessary complexity.

There's no way to cache the individual PackFile that was loaded, because it
isn't kept separate. There's no way to know what Subs were tagged with what
pragmata, because the flags are cleared. The only way you have of protecting
yourself from harmful collisions is a patchwork of quick fixes and magical
coupled behavior. Libraries avoid re-initialization not because we are
careful to make initialization a separate and explicit operation, but because
we attempt to magically and automatically re-initialize things all over the
codebase, clearing flags in otherwise immutable data structures.

If you think about it hard enough, you might start to see why I've been so
energetic and enthusiastic about getting some of this crap cleaned up.

Of course, some of the problems and idiosyncracies are the reasons why an
upgrade path is neither going to be easy nor straight-forward.

First things first, we need to start replacing `load_bytecode_s` with
`load_bytecode_p_s`. That deprecation notice is already in, although it's not
a straight-forward or easy upgrade. Remember that `load_bytecode_s`
automatically initializes the bytecode by executing `:load` Subs. The new
op does not automatically initialize, and does not not keep track of what
has been initialized, or how. Separation of concerns is a very good thing in
general, but in this case we do need to create a new mechanism to replace this
behavior before we can start to move forward. I haven't quite decided on a way
to do this yet, but I'm looking at several options. What is most likely is to
add a cache to the PackfileView PMC type of arbitrary user data. That way we
can get a reference to the PackfileView, check if it has been initialized yet,
and initialize it if not.

The next thing we need to do is rip out all the calls to `do_sub_pragmas` and
all the calls to the subs which call it, directly and indirectly. Library
initialization becomes explicit and performed in PIR code, not implicitly
performed throughout the C code base. We *can* add a wrapper that does it from
C code to ease the transition, but we lose all our performance gains if we go
that route. One place where we currently *need* to call `:init` Subs from C
is during program execution from the C frontend. Of course, we only need to do
it that way until we have an alternate frontend which is written in PIR.

If we have a frontend that bootstraps to a PIR entry function earlier, we can
do packfile loading and packfile initialization from PIR code. This way we
can avoid overhead of creating multiple runloops before we've even started
executing our program. In the `whiteknight/frontend_parrot2` branch I've
started working on this very thing. The only stumbling block I am running
into is finding a good and inexpensive way to separate out the arguments which
need to be handled in C from the arguments we can safely handle from PIR, and
then handling those arguments in the correct places. This isn't technically
difficult, I just need time (and feedback!) to make sure I do it in a sane,
performant, and maintainable way.

Right now, in the `whiteknight/imcc_tag` branch, the two flags `:load` and
`:tag("load")` function almost identically (same with `:init` and
`:tag("init")`), except the second is not found or executed automatically by
`do_sub_pragma()`. Once we have the new frontend in place, I can make the
switchover so that `:load` internally is treated exactly the same as
`:tag("load"), and we can get rid of that portion of `do_sub_pragma()`
completely. At that point, we can slowly phase out the `:load` and `:init`
syntax.

To recap, here's the basic road forward:
1. Get the basic `:tag()` syntax well-tested and merged into master. This
   should be non-destructive and most people won't even notice it has been
   added. People who want it can start to make immediate use of it. Plobsing
   has also suggested some further possible performance improvements here, so
   we can be making it even better.
2. Implement some kind of a cache for keeping track of which libraries have
   been initialized, and in which ways.
3. Start moving towards a new frontend, which bootstraps to PIR as early as
   possible.
4. Remove `load_bytecode_s`
5. Update `:load` and `:init` to be thin aliases over `:tag("load")` and
   `:tag("init")` respectively. As far as the user of PIR code is concerned,
   this step should be completely transparent.
6. Rip out much of the `do_sub_pragma()` logic.
7. Deprecate and remove `:load` and `:init` flags.
8. Try to unify even further, switching `:main` to `:tag("main")` (which
   should be both simple and possibly have other benefits) and `:postcomp` to
   `:tag("postcomp")`. Other flags like `:immediate` and `:anon` will have to
   be re-thought and possibly removed in later, separate refactors.
9. Profit!

It's a long way forward, but there's a very good chance that most of this
could be done before the 4.0 release in January, and the remainder shortly
thereafter.

