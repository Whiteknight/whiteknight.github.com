In my last post I mentioned my new `kill_sub_flags` branch, where I'm finally
ripping out the old `:load` and `:init` flags from IMCC and updating everything
to use the new `:tag()` pragma, the new `load_bytecode_p_s` opcode and the new
PackfileView PMC type. Combined, these new mechanisms promise to be more
flexible, more powerful, more generally useful and have better performance than
the old versions. Considering I haven't done a lot of parrot core development
in the past two or three months, I figured jumping right in with such a large
project would be a great way to hit the ground running.

So what have I accomplished so far, besides awkwardly mixing metaphores? A lot,
actually. In the branch at the time of writing this post I've done the following
things:

1. Removed `:load` and `:init` flag syntax from IMCC
2. Removed `Parrot_load_bytecode` and `Parrot_load_language`, replaced with
   the `Parrot_pf_load_bytecode` and the new `Parrot_pf_load_language`
   functions.
3. Fixed pbc_to_exe so generated executables properly merge and track tags
4. Fixed pbc_merge to properly merge tags
5. BONUS! I've fixed pbc_merge to also properly merge annotations
6. Updated much of the runtime library to use the new `load_bytecode_p_s` and
   `load_language_p_s` opcodes
7. Added several new convenience methods to the embedding API and the packfile
   API, especially dealing with annotations and tags
8. Removed auto-compile behavior. `load_bytecode` and `load_language` ops now
   only load packfiles and don't assume that a non-packfile is a PIR file.
9. Cleaned various bits of code in the packfile system, IMCC, pbc_to_exe and
   pbc_merge

I've still got a little bit of work to complete before we can even talk about
wider testing and merging:

1. I'm getting a weird compile error in the `ncurses.pir` compilation that I
   haven't tracked down yet.
2. I have a few more bits of the runtime library to update, so we can complete
   the build.
3. I need to update Winxed and NQP to not use `:init`, `:load` or
   `load_bytecode_s`, and any other projects I can find that rely on the old
   syntax.  I've done a fast update of the winxed and nqp snapshots in the core
   repo, but I need to submit patches for those projects directly so future
   versions are also compatible.
4. There's always more cleaning to do in the packfile subsystem, IMCC and
   elsewhere. This is open-ended and I won't hold up the branch just to do
   cleanup busywork.

One of the things I've noticed is that it's very annoying to have to copy+paste
the same bytecode loading sequence into all the library files that were
previously using `load_bytecode_s`. It's about 10 lines of PIR code (4 or 5
lines of Winxed or NQP), but is still annoying to copy+paste anything. I really
have been wishing that we had some kind of mechanism for providing a simple
pbc-based runtime in libparrot itself. I could create a fallback mechanism in C
that finds and executes `:tag('load')` or `:tag('init')` Subs but that would
create nested runloops and defeat all the performance improvements (not that
they are large, but every little bit counts).

Of course, figuring out how to provide such a basic common runtime and then
trying to figure out what kinds of logic to put there would be another big task
and one I don't really want to tackle here.

The real solution, I think, is to rewrite much of our runtime library in Winxed
or NQP. Once those compilers are updated to have modern bytecode loading
semantics and tools, we'll be able to get those things for free in the various
libraries where they are used. That's another huge problem that I'm not ready to
tackle in this branch either. It might make a great source of work for GCI
students this coming winter, and might also be fodder for some new library
components to add to PACT or Rosella too.

