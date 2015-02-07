---
layout: post
categories: [Parrot]
title: Problems with Sub
---

As a Winxed user, I haven't made a heck of a lot of use of Parrot's MMD
features. I've used it in NQP, but the details are sufficiently abstracted in
that language that you don't really get the feel for what is occuring at the
lower levels. Since the feature is so messy, I've made some effort to avoid
using it at the PIR level. Let me rephrase that. I've made some effort to
avoid PIR entirely.

As I mentioned in a previous post, I've been [working on adding MMD support to
winxed][mmd_winxed_post]. To really get a handle on multiple dispatch, I did
what I always do: I went right to the source. I opened up the MultiSub PMC,
which is the primary user-visible entry way into the multiple dispatch system.
What I found there was...underwhelming. The MultiSub PMC is not descended from
the Sub PMC. It's basically an array which does basic type-checking on insert
operations (push_pmc, set_pmc_keyed_int, etc) to ensure that objects being
added to the array are indeed Sub PMCs. Actually, it wasn't even consistent,
some of the insert vtables checked whether the PMC was a Sub, other insert
vtables checked whether the PMC satisfied the "invokable" role. While similar,
the two checks will allow different PMCs. I found several vtables that were
redundant and unnecessary, and I found a few other problems as well.

[mmd_winxed_post]: /2011/08/15/multiple_dispatch_in_winxed.html

Combine that with what I know about the shortcomings of the Sub PMC, and
nasty code in the associated subsystems (`src/sub.c`, `src/multidispatch.c`,
etc), and I think we have a major problem on our hands. In this post, which
could potentially turn into a long series of posts, I'm going to talk about
some of the problems with Subs and the way I plan to fix them. MultiSub is
one of the pieces that is going to come along for the ride.

I'm planning to make several fixes to the systems I talk about below, although
exactly what I am going to fix and how I am going to do it are still up in the
air. Feedback and suggestions, as always, are appreciated. I know that what we
have is bad enough to need fixing, even if I don't currently know all the best
ways to proceed.

Want me to prove to you that we have a major problem? Here is the complete,
unadulterated list of attributes for the Sub PMC:

    ATTR PackFile_ByteCode *seg;        /* bytecode segment */
    ATTR size_t             start_offs; /* sub entry in ops from seg->base.data */
    ATTR size_t             end_offs;
    ATTR INTVAL             HLL_id;         /* see src/hll.c XXX or per segment? */
    ATTR PMC               *namespace_name; /* where this Sub is in - this is either
                                             * a String or a [Key] and describes
                                             * the relative path in the NameSpace */
    ATTR PMC               *namespace_stash; /* the actual hash, HLL::namespace */
    ATTR STRING            *name;            /* name of the sub */
    ATTR STRING            *method_name;     /* method name of the sub */
    ATTR STRING            *ns_entry_name;   /* ns entry name of the sub */
    ATTR STRING            *subid;           /* The ID of the sub. */
    ATTR INTVAL             vtable_index;    /* index in Parrot_vtable_slot_names */
    ATTR PMC               *multi_signature; /* list of types for MMD */
    ATTR UINTVAL            n_regs_used[4];  /* INSP in PBC */
    ATTR PMC               *lex_info;        /* LexInfo PMC */
    ATTR PMC               *outer_sub;       /* :outer for closures */
    ATTR PMC               *eval_pmc;        /* eval container / NULL */
    ATTR PMC               *ctx;             /* the context this sub is in */
    ATTR UINTVAL            comp_flags;      /* compile time and additional flags */
    ATTR Parrot_sub_arginfo *arg_info;       /* Argument counts and flags. */
    ATTR PMC               *outer_ctx;       /* outer context, if a closure */

Have you gone cross-eyed yet? Are you as infuriated by this as I am? If not,
continue reading. If so, continue reading for the lulz.

Here's a question for you: How does Parrot implement closures? Parrot
implements closures by taking a Sub, cloning it, and setting a pointer to the
parent Sub's active CallContext in the `->outer_ctx` field of the child Sub.
In other words, a Closure is not it's own type of thing. It's a Sub, but with
one extra field set in it. Closures are basically ordinary Subs except for one
detail: A closure has an outer lexical scope which it can search through to
find values of lexical variables. Why every Sub needs to include that
ability is beyond me. Closure should be either a subclass of Sub or, if we
want more flexibility, it should be a mixin.

What's the difference between an ordinary Sub and a vtable override? Well, the
vtable override has an index value set in `->vtable_index`. What if we have a
single Sub that we would like to use for two separate vtable slots? What if we
have a single Sub, for something like `set_pmc_keyed` and `set_pmc_keyed_str`,
and we want PCC to automatically coerce arguments from string to PMC to share
a single implementation? The result is major fail. It simply doesn't work.
It's a reasonable idea, but Parrot absolutely does not and *can not* support
it. At least, not right now.

Let me ask you another question: What is the `namespace_stash`? And, more
importantly, why does the Sub need to know where or how it's being stored?
Keeping track of it's own contents is the business of the NameSpace PMC, not
the job of the Sub PMC. What if we want to reference a single Sub from
multiple namespaces? Or, what if we want to reference a single Sub by multiple
names within a single namespace? What if we want the Sub not to be
automatically stored in *any* namespace at all? Suddenly, `namespace_name`
isn't looking too smart either. If your answer to any of these questions above
involves cloning the Sub PMC, that answer is just wrong. Why should we have to
clone a Sub, just so we can store a reference to it in two separate places?
It's not the job of the data to keep track of the container, it's the job of
the container to keep track of the data.

Similarly, isn't it the job of the MultiSub to keep track of the Subs and
their corresponding signatures? I mean, what if I have this Sub:

    .sub 'Foo'
        .param pmc all_args :slurpy
        ...
    .end

...And I want that Sub to be called from a MultiSub for multiple different
signatures, only redirecting certain variants to specific alternatives? If the
MultiSub were some sort of hash or search tree instead of a dumb array, and if
it kept track of the signatures associated with each Sub instead of asking the
Sub to keep track of it's own signature, we gain all that flexibility. Also,
I suspect, there are performance wins to be had if we break a signature
key up into a search tree or search graph and traverse it instead of doing an
in-place manhattan sort on sig lists every time we call the MultiSub. It's
absolutely absurd that when you store a Sub in a MultiSub, the *Sub tells the
MultiSub how to store itself*. How untenable and unmanagable is it, in the
long run, to have a system where values tell the containers they are stored in
how they need to be stored and organized? Very, that's how much.

Basically, I'm saying we should change this:

    push multi_sub, my_sub

To this:

    multi_sub[signature] = my_sub

The user can pick the signature, and can reuse a single sub for multiple ones.

When you compile a PIR file with IMCC, IMCC collects all the relevant
information together and jams it all into a single place: The Sub. When Parrot
loads in a packfile, it reads each Sub entry, uses the namespace information
therein to recreate the NameSpace tree, and inserts Subs into the proper
namespaces. Then, when we create a class, the Class PMC searches for the
NameSpace with the same stringified name, and pulls all the methods out of it
Keep in mind that
[namespaces aren't supposed to hold methods at all][namespace_cleanups], so
the list of methods in the namespace has to be kept separate and hidden until
the class (and only the Class) asks for it. At that point, since the NameSpace
is itching to dump off the responsibility, it deletes its own copy of the list
as soon as it is read. We insert things into the NameSpace that don't belong
there, and we ask the NameSpace to carefully ignore some things, and store
other things but to do so in a secret, hidden way. Awesome!

[namespace_cleanups]: /2011/06/22/namespace_cleanups.html

Similarly, when Parrot loads in a packfile and inserts Subs into the
NameSpace it's the job of the NameSpace to automatically and invisibly insert
Subs with similar names and the `:multi` flag set into new MultiSub
containers. The Sub tells the NameSpace how the NameSpace must store the Sub,
under which names, and in which locations. Then if there's a `:multi` involved
the Sub tells the MultiSub how to do it's job too. The Sub sure is bossy, and
even if you're a fan of centralized control in a generalized philosophical
way you have to admit that the results here are...less than spectacular. If
you set a Sub with the same name but without the flag set, the NameSpace
overwrites the old one. But if the flag is set, the two are merged together
into a single MultiSub. So here's yet another question for you:

    # What happens here?
    my_namespace["foo"] = $P0

In this short example, assuming we don't know where `my_namespace` comes from
or what it previously contains, what happens? Luckily we have some easy rules
to follow to figure this out:

1. If `$P0` is any type of PMC *except* a Sub, a MultiSub, or a NameSpace,
   it's stored as a global overwriting any existing global by the name
   `"foo"`.
2. If `$P0` is a user-defined subclass of Sub, it's treated differently in
   ways I don't seem to understand. The code is there, but when I try to trace
   it, I weep.
3. If `$P0` is a Sub with the `:method` flag, it will be stored in a separate,
   secret hash of methods, to be added to the Class of the same name when the
   class is created. *UNLESS* the type in question is a built-in type with a
   PMCProxy metaobject instead of a Class metaobject, then the exact sequence
   of events is mysterious and uncertain, because built-ins can be
   instantiated and used *before* the associated PMCProxy is ever created, so
   there is no single way to fetch all the methods from the NameSpace at once.
   I think the implementation of the Parrot `default` PMC automatically looks
   in the namespace whether the PMCProxy has been instantiated or not. I don't
   know the details, and I really don't want to know.
4. If `$P0` has the `:multi` flag set, it will get merged into a MultiSub,
   together with a previous Sub of the same name, if any. Unless the previous
   entry is not also a `:multi`, then it overwrites. If there is no existing
   MultiSub PMC or any value of the same name, a new MultiSub PMC is
   automatically created for it.
5. If `$P0` has the `:vtable` flag set, it will also get stored away in a
   super-secret location, to be grabbed by the Class when necessary, with all
   the same caveats as I mentioned for the `:method` flag, above.
6. If `$P0` is a `:method` or a `:vtable` with the extra `:nsentry` flag set,
   Then it *is* stored in the namespace anyway, in addition to being stored in
   a way that is fetchable by the Class or PMCProxy.
7. If `$P0` is a NameSpace, it's stored in the `my_namespace` as a child, and
   becomes a searchable part of the NameSpace tree in a way that does not
   interfere with a non-namespace object of the same name, if any. The exact
   mechanism for doing this involves creation of large numbers of unnecessary
   GCable PMCs, and the tears of children.

This all sounds like the best, most well-thought-out, best designed and best
implemented solution, doesn't it? And there isn't a hint of magic or confusion
anywhere in sight.

All of those things, every last bit including the problems with the Sub PMC
containing too many unnecessary attributes, are all symptoms. The single
underlying problem that necessitates all of this crap is that the packfile
loader automatically creates NameSpaces and automatically inserts Sub PMCs
into them when a packfile is loaded. When you jam a bunch of stuff into the
NameSpace automatically, without consideration for where it really belongs,
you're forced to insert a bunch of logic inside the NameSpace to deal with it.
Take that away, force the packfile loader to stop jamming data where it
doesn't belong, and suddenly all the crap I mentioned above goes away. Piles
and piles of the foulest, most garbage-ridden code I have ever seen evaporates
away into a fine mist of unicorn farts. I say good riddence.

So what's the alternative? Well, 6model doesn't use the NameSpace as
intermediate storage for methods. When you create a class with 6model, you
get individual references to the Subs you want and you insert them, by name
and static reference, into the Class. Reuse the same Sub as much as you want.
We can extend this idea even further too, by applying it to MultiSubs. If you
want a MultiSub, create one yourself and insert the functions you want into
it. For that matter we can extend the idea all the way to NameSpaces
themselves. Parrot shouldn't automatically create or populate any NameSpace
PMCs. None. Not ever. The user can create and populate themselves. For all
these things we can either do the creation at runtime, or we can do the
creation at compile time and serialize the whole Class/MultiSub/NameSpace into
the packfile as well. If we don't want it, Parrot won't force it upon us
automagically.

HLLs like Winxed or NQP-rx or anything else can be modified to generate the
necessary instantiation code in the generated PIR output, instead of relying
on Parrot to do it for us. There's a good chance that this approach could be
more performance-friendly, because we would do less moving of data, less
calling of methods on startup and packfile loading, and less of other
unnecessary operations as well.

I think this sounds like a much better system, personally, and it's a
direction I want to start working towards. The ultimate goals are as follows:

1. Sub PMC should be slimmed down to be a simple wrapper around a section of
   bytecode. It should contain a little bit of information necessary to be
   invoked and to execute bytecode only. It shouldn't hold much other
   information. It probably does need to include a pointer to a LexInfo PMC
   for lexicals, but that's it. Maybe a subclass or a mixin can hold the
   pointer to LexInfo, but I don't want to push it.
2. Other behaviors of Sub, like Closures and Methods can be broken out into
   separate subclasses or mixins, if such is warranted. For Closures I think
   we definitely want it. For Methods, it might be unnecessary. I'm not really
   convinced that Methods want to be treated any differently from ordinary
   Subs, but maybe HLLs think differently. Of course, maybe HLLs can provide
   subclasses for Methods themselves.
3. Class, NameSpace, and MultiSub should all be given improved interfaces for
   populating their contents at runtime. NameSpace in particular needs to be
   dramatically cut, sliced, and refactored to remove all the magic and
   garbage (and magical garbage). NameSpace should be, essentially, little
   more than a Hash with a name and the ability to recursively search
   contents using multi-part keys.
4. IMCC needs to be changed to remove code that fills Sub full of unnecessary
   information. Also, if we have time, IMCC needs to be deleted.
5. The MMD system is going to need to be re-thought. Specifically, I don't
   like the idea of keeping around a bunch of signature lists and needing to
   sort them on every invocation. Or, having to do it every time by default.
   Instead, I think wins can be had if we use some variety of a search tree
   instead. Also, we don't assume every Sub is associated 1:1 with a single
   signature. The MultiSub should handle mapping a signature to a Sub, but
   should be flexible enough to support an N:1 mapping.

What I want to impress upon you, the reader, with this post is that the Sub
PMC is extremely poorly designed. Since it's the lynchpin of Parrot, the thing
that makes control flow work, that's a pretty bad thing to get so horribly
wrong. I don't want to say that I have all the designs and solutions to fix it
just yet, but this issue is squarely in my crosshairs right now and you can
expect some movement on this issue in the near future.


