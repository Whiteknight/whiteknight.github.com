---
layout: post
categories: [Parrot, PCC]
title: VTABLE Morph and Context Reuse
---

Here's a function from the internals of Parrot's calling conventions (PCC)
subsystem. It is one of a handful of similar functions for creating a new
CallContext PMC in preparation for a Sub invoke (function call). This
particular variant takes a list of arguments from the variadic argument list
passed to the C function:

    void
    Parrot_pcc_set_call_from_varargs(PARROT_INTERP,
            ARGIN(PMC *signature), ARGIN(const char *sig),
            ARGMOD(va_list *args))
    {
        ASSERT_ARGS(Parrot_pcc_set_call_from_varargs)
        PARROT_ASSERT(PMCNULL != signature);
        VTABLE_morph(interp, signature, PMCNULL);
        set_call_from_varargs(interp, signature, sig, args);
    }

The details of this function aren't really important, but one thing that
should stand out from even a cursory reading of the function is the use of
`VTABLE_morph()`. What is that, exactly?

The `morph` vtable was originally designed to aide in data dynamicism. You
could switch a PMC from one type to another type in-place, without needing
to return a new PMC and update references. Ignoring the fact that for such
a system to be usable in a broad way we would need about `N^2` morphing
routines from most data types to other data types (or a single, overused,
error routine to tell that the two types aren't compatable). The first
argument to `morph`, like all vtables, is the interpreter. The second argument
is the target PMC, and the third argument is the Class PMC or other Metaobject
to transform the target into. So why is the target type `PMCNULL` here? What
would that operation even mean?

The thing is that `morph` isn't really used much. It's too much of a pain
to implement in any meaningful way, and most projects get along just fine
without it. In fact, Parrot's core internals don't even use it much, except
in implementing a few bits of old legacy logic in our Perl-derived basic
data types: Integer, Float and String. Beyond that, `morph` is basically
unused.

Here's a problem that needs to be solved: Parrot uses
continuation-passing-style (CPS) for control flow, which means that a
function call is *almost* exactly the same as a function return. Both of them
are invocations of some code object. The first is an invocation of the Sub
object and the second is an invocation of the return Continuation object. We can
pass arguments in both directions, which is how Parrot is able to return
multiple values from a single function with ease:

    :(var x, var y, int a = 4, var z [named("Zee")]) = foo();

The trick is that the return is just an invoke, so we can pass arguments and
process them through the exact same code path as we took to call the function
in the first place. Every flag and argument combination that works in a Sub
call will also be usable in the Sub return. The system isn't completely
symetric, but it's pretty darn close.

The big problem here is that since both call and return use the same
mechanism, both of them require a CallContext PMC to be created and populated.
That's two CallContexts per sub invoke, for every single sub in the entire
program. This, to put it mildly, puts a lot of pressure on GC and our memory
subsystem.

So let's get to the point. We need a way to reuse a CallContext object, so that
a return from a function uses the same CallContext as was used to call the
function in the first place. This cuts GC pressure from PCC calls down
significantly. To do that, we need some kind of quick mechanism that we can call
to reset the CallContext, without recursing into a new context like a method
call would do. Therefore, we need either a VTABLE or some other kind of
interface routine somewhere. The list of vtables is already rather limited, and
among them the number of vtables that are both lightly used (so we don't run
into all sorts of semantic conflicts) AND which CallContext doesn't already
implement is more limited still. That's where VTABLE morph comes in. According
to the documentation it "morphs" a CallContext from a Sub invoke into a
CallContext suitable for the return invoke, even though it doesn't morph so
much as just reset some internal state. Here, for your viewing pleasure, is the
code of CallContext.morph:

    /*

    =item C<void morph(PMC *type)>

    Morph the call signature into a return signature. (Currently ignores
    the type passed in, and resets the named and positional arguments
    stored.)

    =cut

    */
    VTABLE void morph(PMC *type) {
        Hash     *hash;

        if (!PMC_data(SELF))
            return;

        SET_ATTR_short_sig(INTERP, SELF, NULL);
        SET_ATTR_arg_flags(INTERP, SELF, PMCNULL);
        SET_ATTR_return_flags(INTERP, SELF, PMCNULL);
        SET_ATTR_type_tuple(INTERP, SELF, PMCNULL);

        /* Don't free positionals. Just reuse them */
        SET_ATTR_num_positionals(INTERP, SELF, 0);

        GET_ATTR_hash(INTERP, SELF, hash);

        if (hash) {
            parrot_hash_iterate(hash,
               FREE_CELL(INTERP, (Pcc_cell *)_bucket->value););
            Parrot_hash_destroy(INTERP, hash);
            SET_ATTR_hash(INTERP, SELF, NULL);
        }
    }

When I say above that a return is *almost* exactly like a call, I mean that they
should be the same but aren't because there's too much magic. Without getting
into too many details, when we make a call the first time Parrot creates a new
CallContext to hold arguments for the call. However when we return, Parrot
attempts to reuse (and morph) the existing CallContext to handle the return. The
only functional difference between a call and a return in Parrot, the only thing
requiring two sets of opcodes and preventing full CPS symmetry, is the fact that
Parrot tries to automatically provide the CallContext for us instead of taking
an explicit CallContext as an argument to the call sequence.

In other words, if we turn this:

    set_args <args>

...into this:

    ctx = get_new_context
    set_args ctx, <args>

And if we turn this:

    set_returns <returns>

Into this:

    ctx = get_return_context
    set_args ctx, <returns>

We've made the system fully symetric, pluged a hole through which bugs may flow
(especially concurrency- and optimization-related bugs), and removed a bit of
unnecessary magic that prevents us from having a simpler, faster, more usable
system.

The best part about making these kinds of changes is that users don't write the
`set_args` and `get_params` opcodes and their friends directly. Instead, you
write something like this in PIR:

    ($P0, $I0) = 'foo'(1, 2, 3)

...and IMCC outputs the correct args for you. If we add better args and patch
IMCC to use those instead, the change will be transparent to most existing users
and will allow more advanced use cases for people who want them.
