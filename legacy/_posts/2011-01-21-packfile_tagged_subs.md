---
layout: post
categories: [Parrot, PIR, Packfiles]
title: Tagged Subs in PIR
---

There has been a certain amount of confusion recently about the PIR subroutine
flags `:init`, `:main`, and `:load`. Certain people, myself included, were
tryting to deprecate the `:load` flag, while other people (specifically people
from the Rakudo project) were suggesting that perhaps we needed more. I've
been thinking about this a lot and have an idea for a nice flexible option
that should satisfy the needs of our users and help us clean and consolidate
the codebase at the same time.

The core idea of tags like `:init`, `:main`, and `:load` is that we want to be
able to attach metadata to subroutines in PIR (and, by extension, other
languages), and be able to execute subs with the given tag in certain
situations. When we use the `load_bytecode` op to load a bytecode file as a
library, for instance, we want to trigger all `:load` functions, in any
order. When we load a bytecode file as an executable, we want to trigger all
the `:init` functions in any order, followed by a single `:main` function.

Fast forward a few months and assume that we now have a working
[PIR compiler PMC type][pirpmc], which we can attach methods and data
attributes to. I'm going to execute this little snippet:

    pircompiler = compreg "PIR"
    $P0 = pircompiler.'compile'(source_filename)

[pirpmc]: /2011/01/18/imcc_interface_functions.html

What happens in this situation? Or, more importantly, what should happen?
We know that the `$P0` PMC is going to be a [PackFile PMC][packfilepmc], but
we don't know what the user intends to do with it. We don't know if the
intention is to take that packfile and immediately (and without sideeffects)
write it out to a new .pbc file, or if the user intends to load it in to
Parrot's interpreter and immediately start calling functions from it like a
library. Maybe the user wants to treat this as a program, and immeduately jump
into its `:main` method. Or, maybe the user wants to introspect the packfile,
to calculate complexity metrics. Maybe the user wants to run an in-place
optimizer over the packfile. We simply don't know what the user wants to do.

[packfilepmc]: /2011/01/17/packfile_and_imcc_branches.html

Assuming that the user wants to treat this like a library and trigger all the
`:load` functions is going to be wrong in many cases. Assuming that the user
wants to treat this like an executable and trigger `:init` and maybe `:main`
is frequently also going to be wrong. And when Parrot makes these kinds of
assumptions we're going to have users who have difficulty with it and need to
implement ugly workarounds to get what they want without Parrot doing too much
without asking. It's my opinion that Parrot should never do what you don't
want it to do without asking first. It's also my opinion that Parrot is tool,
a library for creating a dynamic language runtime, not the end program in
itself. The programmer uses it as a base to do what is needed; Parrot doesn't
always know what that should be.

What we really need is a way for the user to specify exactly what they want
to happen and when. It's not for Parrot to decide. `:init`, `:main`, and
`:load` should be triggered *on command*, not automatically. Here's an
example:

    packfile = pir_compiler.'compile'(filename)
    packfile.'trigger_load_functions'()

Or:

    packfile = pir_compiler.'compile'(filename)
    packfile.'trigger_init_functions'()
    main_sub  = packfile.'get_main'()
    main_sub(main_args)

Suddenly users have control over what executes when. However, this really
isn't a great solution either. For starters, it's ugly to have two methods to
perform such similar actions. Further, it doesn't make any sense to
artificially limit the types of tags that we have. Some compilers need more.
I know Rakudo does. So, maybe we want something like this, borrowing some
terminology from Rakudo:

    packfile = pir_compiler.'compile'(filename)
    packfile.'trigger_phasors'("load")

But then, we want to have any arbitrary phasor attached to a sub. We should be
able to give it any name and attach as many as we want. We may have many
needs. Here we will create a little [PIR frontend program][parrot_in_parrot]
to load a bytecode file, trigger the `:init` phasors, the `:main` function
and a new `:end` phasor for post-facto cleanup:

    packfile = pir_compiler.'compile'(filename)
    packfile.'trigger_phasors'("init")
    main_sub  = packfile.'get_main'()
    main_sub(main_args)
    packfile.'trigger_phasors'("end")

[parrot_in_parrot]: /2011/01/20/parrot_in_parrot_new_frontend.html

What do these new flags look like in PIR? Here's one idea:

    .sub foo :tag("init")
    .end

Right now, Parrot has a really ugly system for dealing with `:init` and
`:load` flags. When we want to fire all of the `:init` subs, for instance,
we loop over all PMCs in the packfile's constant table looking for subs. When
we find a sub, we check its flags. If the flags match what we are looking for,
we execute the sub and then remove the flags. The current process for firing
`:init` and `:load` is destructive, which means that once you do it you can't
do it again. This doesn't come up much, but it's still an ugly restriction.

Some people would probably argue that we don't need to be adding a million new
methods to the PackFile PMC. The [single responsibility][srp] of the PackFile
PMC should be working with the PackFile structures and file format, not
executing subroutines. Especially not assuming to know how those subroutines
should be executed. In answer to that I suggest maybe we could do something
like this:

      $P0 = packfile.'get_tagged_subs'("init")
      $P1 = iter $P0
    trigger_init_top:
      unless $P1 goto trigger_init_bottom
      $P2 = shift $P1
      $P2()
      goto trigger_init_top
    trigger_init_bottom:

This is a little bit more work, but not a whole lot. Also, it would be trivlal
to encapsulate this logic into a subroutine and reuse it for all your phasors.
The benefit of this situation is that the packfile is only responsible for
providing read access to the list of subs, the user can decide whether to
execute each and if so, how. For the common case, this code would be hidden
away in the [Parrot executable frontend][pir_frontend].

[pir_frontend]: /2011/01/20/parrot_in_parrot_new_frontend.html

We also start to get the idea that maybe these phasor subs could start to take
arguments, or return results. I can think of at least a handful of reasons why
I would like to have that feature available in a PIR-based Parrot frontend
program like I was envisioning a few days ago. Other languages, such as Rakudo
would have their own entrypoint routine and would be able to decide when, if
and how to trigger each of it's tagged sub types.

One criticism I can forsee is that different libraries and executables may
use different sets of tags that are expected to be executed in different
orders or with different parameters than the "normal" set. This is true, but
I would counter to say that the way to load and initialize a library is part
of that library's API and should be well documented and understood by users
before use.

The more I think about this idea the more I like it. This should help to clean
up a lot of code in libparrot, especially some code which is very ugly and
buggy. We free up a lot of ugly flag logic in IMCC and in the packfiles system
and replace it with a simple keyword search (and maybe a hash for better
performance). We should be able to resolve a handful of long-standing tickets
and issues, and provide a lot of new flexibility to the users of libparrot.
