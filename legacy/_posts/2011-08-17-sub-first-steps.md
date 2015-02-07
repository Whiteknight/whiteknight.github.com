---
layout: post
categories: [Parrot, PackFile, IMCC]
title: Subs, First Steps
---

Yesterday I posted [something of a rant about the poor state of our Sub and
NameSpace PMCs][sub_problems_post]. The problems in these two PMCs, and
resulting problems caused by them in other places, are really symptoms of a
single problem: That the packfile loader blindly inserts Sub PMCs into the
corresponding NameSpace PMCs at runtime, and leaves the NameSpace PMC to sort
out the details.

[sub_problems_post]: /2011/08/15/sub_problems

I would like to show you the three offending pieces of code that really lead
us down this rabbit hole. The first snippet is from the heart of IMCC, the
code that adds a namespace to a Sub during compilation:

    ns_pmc     = NULL;
    if (ns) {
        switch (ns->set) {
          case 'K':
            if (ns_const >= 0 && ns_const < ct->pmc.const_count)
                ns_pmc = ct->pmc.constants[ns_const];
            break;
          case 'S':
            if (ns_const >= 0 && ns_const < ct->str.const_count) {
                ns_pmc = Parrot_pmc_new_constant(imcc->interp, enum_class_String);
                VTABLE_set_string_native(imcc->interp, ns_pmc,
                    ct->str.constants[ns_const]);
            }
            break;
          default:
            break;
        }
    }
    sub->namespace_name = ns_pmc;

In this snippet, `ns` is a `SymReg*` pointer. Basically, it's a pointer to a
parse token representing the NameSpace declaration. `ns->set` is the type of
declaration, `'K'` for a Key PMC and `'S'` for a STRING literal. The two forms
are written in PIR as:

    .namespace "Foo"
    .namespace ["Foo"]

Actually, I don't know if the first is still valid syntax, so the `'S'` part
of this block might be dead code. We can dig into that later, it's not really
important now. The last line of the snippet sets the `namespace_name`
attribute on the Sub PMC to be either a Key PMC or a String PMC containing the
name of the namespace.

Notice that the Sub PMC has attributes `namespace_name` and `namespace_stash`,
both of which are PMCs. The `namespace_name` is populated at compile time and
is used by the packfile loader to create the NameSpace PMC automatically. A
reference to that NameSpace PMC is stored in `namespace_stash`. Thereafter,
`namespace_name` is rarely used. We can definitely talk about ripping out one
of these two attributes, if we can't make bigger improvements in a reasonable
amount of time. In the long run, both will be gone.

The second snippet I want to show you is inside the packfile loader:

    for (i = 0; i < self->pmc.const_count; i++)
        self->pmc.constants[i] = PackFile_Constant_unpack_pmc(interp, self, &cursor);
    for (i = 0; i < self->pmc.const_count; i++) {
        PMC * const pmc  = self->pmc.constants[i]
                              = VTABLE_get_pmc_keyed_int(interp, self->pmc.constants[i], 0);
        if (VTABLE_isa(interp, pmc, sub_str))
            Parrot_ns_store_sub(interp, pmc);
    }

I've removed comments for clarity (No, that's not some kind of a joke. There
were comments in this snippet, and they are *not* helpful for
understandability).

In this snippet we first loop for the number of PMC constants, unpacking and
thawing each from the packfile. Then in the second loop we loop over all of
them *again*, to see which, if any, are Subs. For any Subs, we call
`Parrot_ns_store_sub` to read the `namespace_name` out of the Sub PMC, create
the NameSpace if necessary, and then insert that Sub into the namespace. That
brings me to my third snippet of code:

    ns = get_namespace_pmc(interp, sub_pmc);

    /* attach a namespace to the sub for lookups */
    sub->namespace_stash = ns;

    /* store a :multi sub */
    if (!PMC_IS_NULL(sub->multi_signature))
        store_sub_in_multi(interp, sub_pmc, ns);

    /* store other subs (as long as they're not :anon) */
    else if (!(PObj_get_FLAGS(sub_pmc) & SUB_FLAG_PF_ANON)
        || sub->vtable_index != -1) {
        STRING * const ns_entry_name = sub->ns_entry_name;
        PMC    * const nsname        = sub->namespace_name;

        Parrot_ns_store_global(interp, ns, ns_entry_name, sub_pmc);
    }

This snippet comes to us from the heart of the `Parrot_ns_store_sub` routine
I mentioned above. This is basically a duplication of some logic found in
the NameSpace PMC. My first thought was that the duplicated code paths should
be merged. My second thought is that they both need to just be deleted. In
this snippet, we call `get_namespace_pmc` to find or create the NameSpace that
suits the current Sub. If the Sub is flagged `:multi`, we add it to a MultiSub
instance in that namespace. Otherwise, so long as the Sub isn't flagged
`:anon` or `:vtable`, we add it to the NameSpace.

So those are the three bits of Parrot that are causing all these problems.
These are the three bits that really encapsulate what the problem with Sub is,
why it's so bad, why Subs eat up so much memory, and why load performance on
packfiles is so poor. These three, small, innocuous snippets of code are
really the root of so many problems. This is all it takes.

What is really being done here? IMCC has all the compile-time information,
and as it's going along it's collecting that information in a parse tree of
`SymReg` structures. A `SymReg` representing a Sub contains a pointer to a
`SymReg` for the owner namespace, a `SymReg` for the string of the Sub's name,
a `SymReg` for each flag, etc. When it comes time to compile the Sub down to
bytecode, it creates a Sub PMC and dutifully stores all this information from
the `.sub` `SymReg` into the Sub PMC. After all, all that information is
together when we parse and compile it, so storing it all together in the same
place for storage does seem like a reasonable thing to do. We store the Sub
PMC in the constants table of the packfile and move on to the next Sub.

On startup then, since we have all these Sub PMCs and each contains all the
necessary data to set themselves up, we read out all this data and start
recreating the necessary details: NameSpaces, MultiSubs, etc. This all makes
some good sense as well, in theory. In practice, we end up with lots of bits
of data (the Subs) tasked with recreating the containers in which they are to
be stored (the NameSpace, Class, and MultiSubs). This means that each Sub
needs to contain enough information to possibly recreate all the possible
containers where it could be stored (NameSpace, Class, and MultiSub), and
that is extremely wasteful.

Here's the big question I have: Why are we creating the NameSpace and the
MultiSub PMCs at runtime? Why don't we create them at *compile time*, and
simply have them available and ready to use already when we load in the
packfile?

Here's an alternate chain of events to consider. When IMCC reaches a
`.namespace` directive and creates the associated `SymReg`, we should also
immediately create a NameSpace PMC. Then when we create `.sub`s in PIR, we can
add each to the current NameSpace, instead of storing the namespace `SymReg`
reference in the Sub. When we go to create the bytecode, we serialize and
store all the NameSpace PMCs, which will recurse and serialize/store all their
component Subs. The same thing goes with MultiSubs: When we find a `:multi`
flag, IMCC should create a MultiSub PMC immediately instead of a Sub PMC, and
perform the mergers at compile time. When we serialize the NameSpace, it
recursively serializes the MultiSubs, which recursively serializes the
various component Subs. One other thing we are going to want to do is create
Class PMCs, or some kind of compile-time stand-in. This way we are able to
handle `:method` Subs easily, by inserting them into Classes at compile time,
not runtime. There are some complications here, so I'll discuss them in a bit.

The real beauty of this change shows up on the loader side: When we load the
packfile, we loop over and thaw all the PMCs. Then....We're done. Maybe when
we load a NameSpace we need to fix it up and insert it into the NameSpace
tree. Also, when we load a NameSpace with the same name as a NameSpace that
already exists in the tree we need to merge them together. That's a small and
inconsequential operation, especially considering how much other code we're
going to delete or streamline.

Here are some problems that we are going to fix:

1. Methods will never need to be stored in the NameSpace, even temporarily
2. Improve startup performance, by not needing to do a `VTABLE_isa` on every
   PMC to see if it's a Sub. If so, we can avoid logic for inserting the Sub
   into an auto-created NameSpace, the storing it in weird ways depending on
   flags, and doing all that other crap.
3. We can significantly cut down logic from the NameSpace PMC, and probably
   make them much cheaper in terms of used memory and logic overhead
4. We can slim down several fields from Sub: `namespace_stash`,
   `namespace_name`, `vtable_index`, `method_name`, `ns_entry_name`,
   `multi_signature`, and probably `comp_flags` too. Some of these are going
   to require deprecations, and some of them are going to require us to
   provide workarounds in certain areas.
5. If we can make IMCC less reliant on direct structure access for setting up
   a Sub, which is a real possibility when there is less "setting up" to be
   done, we can make it more easy to replace Sub entirely with a custom
   subclass at compile time. That's something we've always wanted but never
   been able to do precisely because there were so many direct attribute
   accesses in IMCC.
6. We can move more processing to compile-time and do less of it at load time,
   decreasing startup time for most programs run from bytecode.
7. We can probably simplify a lot of IMCC logic by actually creating the PMCs
   we need directly, instead of creating a bunch of structures which contain
   the information we need to create the things we need later.

The issue of Classes and Methods is a little bit tricky because looming
overhead are the impending 6model refactors. The `:method` flag on a Sub
really performs two tasks: First, it tells IMCC to automatically add a
parameter to the front of the list called "`self`". Second, it tells the
NameSpace to store the Sub, but to do so in a super-secret way that only the
Class PMC can find. The first use I've always felt was nonsensical, especially
when you consider requests to add a new `:invocant` flag for parameters to
manually specify the name of the invocant, and the fact that the automatic
behavior of it creates a large number of problems especially with respect to
VTABLEs. In my new conception of things, NameSpace won't be storing methods
anyway, so the second use of the flag disappears completely. I say we
deprecate it soon and remove it entirely at the earliest possible convenience.

In the near-term, I think the `:method` flag is going to be used to create a
ProtoClass or an uninitialized Class PMC at compile time. Then, when we call
the `newclass` opcode at runtime it will look to see if we have a Class
(or ProtoClass) by that name and if so will return the existing object,
marking it as being "found" to prevent calling `newclass` again on it. That
sequence of events is a little bit more messy than I would like (again, I
would like to avoid the `:method` flag entirely and do class creation either
entirely at compile time or entirely at runtime, but maintaining compatability
with current syntax may lead in a more messy direction). I'll explore some of
these ideas in a later post.

I've created a branch locally to start exploring some of these ideas, and will
push it to the repo when I have something to show. Depending on how this
project lends itself to division, I may end up with many small atomic branches
or one large overwhelming one. Both paths are equally plausible. I'm going to
start playing around with some code, and when I find something that seems like
a reasonable stopping point I'll try to merge it if reasonable.
