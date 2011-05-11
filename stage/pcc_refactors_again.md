---
layout: post
categories: [Parrot, PCC]
title: Ideas for PCC Refactors and Improvements
---

I've been kicking around an idea for how to improve the Parrot Calling
Conventions (PCC). Two years ago we went through and did a major refactor on
that system, unifying all the various code paths to use a single central
dispatch and argument processing mechanism. The new system uses the
CallContext PMC type to aggregate all parameters for a call. The caller always
puts arguments into the CallContext, and the callee always retrieves
parameters from it. Since Parrot uses CPS, the same mechanism is used for
returns as well. This unification was a great step for Parrot, it was an
important way for us to start getting some of our implementation realities in
line with some of the theoretical underpinnings on which Parrot has been
designed, such as unified CPS. The most important part of these refactors was
the conversion to use CallContext objects throughout the code base as the way
to package up and perform invocations. Other changes we made to the system
were all tangential or ancillary, and nothing else is quite so important or
so sacrosanct that we should not be willing to go back and have some second
thoughts.

PCC is a pretty good system, and the use of CallContexts is a huge step
forward from where we were before the refactors started. I don't want to
fundamentally change that aspect of PCC. Back when we did the refactors the
hope was that by unifying call code paths around the CallContext PMC, that we
would be able to identify more hotpaths and perform new types of optimizations
throughout. Unfortunately, we really haven't delivered on this the way we
should have. Part of the reason is that the first refactor was such a huge
undertaking and brought some real pain to our hackers and users alike. So long
as PCC continues to work, we haven't been too keen to jump in and fiddle with
it again.

I think there are plenty of things we can do to improve performance in the PCC
system by simplifying some of the hot codepaths, and making better use of
compile-time information which we currently discard. First I'm going to
explain how things currently work, and then I'm going to talk about what my
ideas are for changing it. This is not an exhaustive list of possible
improvements, just the few that I have in mind right now.

In Parrot, invocations (function, method, exception handler, and continuation)
are a multi-step process. First, we have to lookup the subroutine object. This
can be either a constant value, a Sub stored in a NameSpace, or a method
stored in an object/class. We create a CallContext object by passing in a
signature array with a series of flags and using those flags to insert a list
of arguments into the CallContext. We store the CallContext in the
interpreter, then invoke the Sub object.

On the callee side, we use a signature array along with the CallContext to
unpack the arguments into registers. The unpacking happens in a function
called `fill_params`, which is a huge behemoth of a function. Actually that's
not fair. It's a behemoth in the context of Parrot, where we tend to
appreciate shorter functions. Other projects might not think it's so large in
comparison. `fill_params` uses a series of loops to get the arguments. It
loops over the flags in the signature, and depending on what each flag is, it
pulls the next value out of the CallContext and inserts it into the specified
register.

I have two problems with this system: The first are the signatures, integer
array PMCs stored as constants in the packfile. These are really weird and
suboptimal ways to store the information about the function arguments and
parameters. What is obnoxious about these is that we have to loop over these
signature arrays to pull arguments out of the caller context. However, at
compilation time we already know what arguments to pass, and could easily
generate a custom sequence of argument passing args without ever needing to
use a signature. Instead of a signature array containing data about how to
pass arguments and retrieve parameters, we could simply output a sequence of
ops that did exactly what we needed. If we need a signature, such as in
multi-dispatch, we can generate those lazily on the fly. The second problem is
with `fill_params`, which needlessly loops over the parameter signature to
extract parameters and insert them into the callee context. I dislike this for
exactly the same reason, we have the information we need at PIR compile time
and we can easily generate a stream of custom fetch ops for the specific
things we need instead of generating a signature at compile time and then
needing to interpret that signature at runtime every time we make a call. In
short, we have plenty of information available at compile time, but we discard
it and compress it into signatures. Every time we make a call at runtime,
*every time*, we need to loop over two signatures: one to fetch args from the
caller, and one to deposit params into the callee. Multiply this by two when
you remember that Parrot uses CPS, so every return is also a call (along with
two more signatures). I suggest we shouldn't need *any* signature arrays for
most calls, and I suggest we shouldn't need *any* loops for most calls either.

I've also left out the part where `fill_params` creates temporary PMCs to hold
flags and data while we traverse signatures. We should be able to invoke a
subroutine or method call without generating *any* temporary object garbage.
This will save on GC pressure too, which is a win-win for everybody.

When we make a call in PIR, it typically looks like this:

    (ret1, ret2, ...) = 'function'(arg1, arg2, ...)

This, as our developers should be well aware, is syntactic sugar over a
sequence of lower-level PASM ops. Here's a basic example of what the generated
code looks like. I'm using a PASM-like pseudocode, because I can't remember
exactly what it really looks like, and I want this example to be clear for
non-insiders to understand.

    $P0 = get_function 'function'
    $P1 = get_call_signature
    set_args $P1, arg1, arg2, ...
    invoke $P0
    $P1 = get_return_signature
    get_results $P1, ret1, ret2, ...

Notice that there are a few problems with this. First, notice that `set_args`
generates a CallContext object, but doesn't return it to the user. That object
is stored directly in the interp. We're making the (bad) assumption that a
set_args call is always immediately followed by an invoke operation, and that
there is no possible way that we could get preempted (see also: concurrency).
The same thing is true of the `get_results` opcode, we make the bad assumption
that nothing ever happens between the time the invocation returns and the time
we try to read the returns CallContext values out of the interp. This needs
to get fixed if we want concurrency, and also if we want users to have more
flexibility and control in dealing with invocations. I think we do want this,
because that opens up whole new ranges of optimization possibilities to the
user. Making these changes won't necessarily improve performance by
themselves, however.

As an aside, A call basically consists of three things as far as Parrot is
concerned: An invocable object (like a Sub PMC), the arguments (in the form of
a CallContext PMC), and an optional return continution to jump back to when
the invocation is complete. We should be exposing all these objects directly
to the user, and making calls using two invoke ops:

    invoke sub, callcontext, retcontinuation
    invoke sub, callcontext

Since the invocant to a method call is passed as the first argument in the
CallContext, we don't need other forms of these opcodes to pass in a separate
invocant, like Parrot provides now. However, I wont get too far off track with
this discussion, I can talk about some of the ops we have (and how they need
to change) in later posts if necessary.

The next thing to see in the example above is that the argument-related ops
`set_args`, `get_params`, `set_returns` and `get_results` are all variadic.
In fact, they are the only variadic ops in the entire system and there is more
than a little bit of logic throughout Parrot core that could be simplified if
we got rid of them complete. What we do is this: The signature tells the
number and the types of the various arguments. Each argument is a pointer
stored directly in the bytecode stream. When we loop over the signature, we
dereference subsequent locations in the bytecode stream to fetch the values,
and push them into the CallContext from there. Life will get a little bit
easier for people working on compilers and related tools (disassemblers, code
analyzers, etc) if we don't have these kinds of ops. It isn't a concern for
most users, but then again having better tools can help those people too.

The set opcodes, `set_args` and `set_returns` loop over the signature and
push values from the caller context into the CallContext PMC for the call.
This process is relatively straight-forward, although not without some
inefficiencies. On the fetch side, `get_params` and `get_results`, need to
do much harder work. This is where that `fill_params` function comes in.
`fill_params` loops over the receiver signature, and starts hunting for
values in the CallContext to match the specific needs. `fill_params` needs
to compare the number of passed arguments with the number of expected
parameters and, depending on settings, throw errors. It needs to determine
if `:optional` params get filled or not, and if an `:opt_flag` is provided
(and provided in the exact correct order) that it is set accordingly. It needs
to handle `:slurpy`. It needs to fetch named arguments, determining what to do
if we don't have named arguments that we've asked for and if we have named
arguments that we have not asked for. As if that wasn't complicated enough, we need to
correctly handle `:named :optional` and other combinations. Also,
`fill_params` needs to coerce arguments to the correct types. For instance,
if we pass an integer and expect a PMC, we need to autobox. This is a heck of
a lot of work for this single function, and the results performance-wise are
not exemplary. As I mentioned yesterday, bacek was seeing this function alone
take up 12% of execution time during the Rakudo core.pm build. That's huge.

So what can we do? What should this look like? How can we cut out cruft? How
can we make this faster?

I suggest we should get away from the four do-it-all ops, `set_args`,
`get_params`, `set_returns` and `get_results`. They are all problematic and
they all need to go. Being variadic means they are hard to deal with and tools
which operate on bytecode (disassemblers, debuggers, anaylizers) all need to
add a heck of a lot of logic to deal with them. Also, because they use
bitflags to encode information about the call into an array, calls are harder
for users to control directly and set up manually.

Second, I suggest that we have all the information we need to make an
efficient call at compile time. We shouldn't condense the details down into a
signature, then have to go to lengths to interpret that signature in a large
loop. CallContext is an object with array-like and hash-like interfaces, and
we have opcodes for interacting with arrays and hashes. Since the opcodes call
the corresponding vtables directly, and those same vtables are used in
`fill_params` and elsewhere to put values into and extract values from the
CallContext, there's no speed savings doing it all internally. The only things
we would need are a handful of new ops to handle some of the special cases
like `:optional`, `:slurpy` and `:flat`.

Here's an example call

    ($P0 :slurpy, $P1 :named("foo")) = 'func'($P2 :flat, $P3 :named("bar"))

Let's see what this could look like if we cleaned it all up and used ops
for setting and fetching arguments and results:

    $P8 = get_function 'func'
    $P9 = new 'CallContext'
    set_arg_flat $P9, $P2
    $P9["bar"] = $P3
    invoke $P8, $P9
    $P10 = get_return_callcontext $P9
    $P0 = get_param_slurpy $P10
    $P1 = $P10["foo"]

Notice that not only should this sequence be thread-safe because we aren't
storing the CallContext in the interp temporarily between subsequent opcode
calls, but that it's simple and straight forward. This sequence would be easy
to disassemble and analyze, requires no signature constants from the packfile,
and requires no loops. We know exactly what arguments we want to pass, and we
just pass them. We know exactly what results we want to fetch, so we just
fetch them.

Notice also in the example above that we create the CallContext and pass it
around in user code. We don't manage it and store it internally, which means
the compiler can have opportunities to recycle and reuse them between similar
calls in a loop, for instance. In the current system, there really is no
opportunity to share CallContext PMCs between subsequent calls, so we need to
create them fresh for each one. That's a lot of churn and a lot of GC pressure
for no benefit.

Here's an example where a CallContext could easily be reused between calls as
an optimization:

    for (var data in data_set)
        obj.my_method(data);

In that loop we could have a single CallContext object, with the first value
always set to the invocant `obj`, and simply replace the value of `data` on
each iteration. That could amount to huge savings, including far less GC
pressure than what we currently do. Of course, it would be up to the HLL
compiler to detect these situations and fold the code appropriately if wanted,
but it's not even possible to do now.

Here's another quick example of some of my proposed changes, from the callee
side:

    .sub foo
        .param pmc bar :optional
        .param int has_bar :opt_flag
        ...

We can convert this into something like:

    .sub foo
        $P0 = get_current_context
        bar = $P0[0]
        has_bar = exists $P0[0]
        ...

Again, for this short example we don't need any special new ops or new
functionality, we just use the existing ops and vtables on the CallContext
in ways we already support. As we saw in the example above, we no longer need
any kind of a signature PMC, we don't need any loops. We know exactly what we
want to get and we go get it. This version should not be any slower than what
we currently have with `fill_params`, and might be a good deal faster if we
can cut out extra GC churn.

As a consequence of this, a lot of the remaining ugly code in the PCC
subsystem involved in creating, examining, and looping over signatures can
disappear. Those are ugly holdovers from the "before time", and there's no
reason we need to continue cargo culting them around in Parrot. `fill_params`
and supporting functions can disappear because they are no longer necessary.
Combine this with a few other changes to returns and removing some special
cases there should bring even more benefits.

Consider also the case of multiple dispatch. If parameter and argument lists
are not fixed in constant signature arrays, the user has much more control
over what arguments to fetch, and in what order. The user could choose not to
fetch certain arguments at all! Consider a function which takes a string
command, and depending on the value of the command takes a variable-length
sequence of additional parameters. Currently, the way to do that might be to
use multiple dispatch to choose between candidates with different argument
lists, and in each one we would check that the value of the command is
consistant with the received argument list arity. MMD comes with its own
overhead, so this is only going to make performance worse. Instead, the user
can read the first parameter, then decide what additional parameters to read,
if any. We don't have to fetch parameters we don't need, and if we are really
lazy about it we can avoid fetching certain parameters at all, in some
situations.

In short, a few points:
1. Expose the CallContext vtable interface directly to the user, for more
   flexibility and control
2. Get rid of signature strings, loops, and `fill_params`
3. Use existing opcodes, where possible, to interact with the call context.
   Create new opcodes as needed.
4. Kill the variadic opcodes for setting and fetching params. They're ugly and
   inefficient.

Basically, I want to give the user more control and improve performance by
deleting code and creating far fewer PMCs for signature arrays and far fewer
temporaries during PCC dispatch. It's a win-win as far as I can see.

How do we get there?

The hard part is getting there. Right now we would need to modify IMCC to
output a new sequence of argument passing and parameter fetching ops, in place
of the old variadic opcodes. This is no small task. I tried making these
changes in a branch as proof-of-concept and the results were not pretty. This
is partially because I still have a lot to learn about the way IMCC works, and
also because when I first tried, I didn't go all out. I tried to preserve too
much of the existing behavior for backwards compatibility and I ended up
having to code circles around where I really needed to be. To embrace this
idea fully, we need to make changes throughout IMCC, including some changes
which (I hope) will simplify IMCC logic quite a bit.

So long as everybody uses the PIR sugar and does not attempt to use the PASM
opcodes directly, the switchover should be mostly transparent from the
perspective of the user. People and tests still using PASM to make calls are
going to be in for a rude awakening (can we kill those old PASM tests
already?) Once the switchover happens, we can start exposing the opcode
functionality directly to the user, and then start removing the superfluous
syntactic sugar for `.param` declarations.

This is an idea for refactors that I sincerely believe has merit and could
bring real performance benefits. I don't think the performance benefits are
going to be gigantic, but then again when you combine decreased PCC overhead
with decreased GC churn, there very well might be real savings to be had.
There are other bottlenecks in PCC dispatch that we will need to tackle as
well which I don't even mention here. However, this is sure to bring at least
some performance benefits and also a number of other benefits as well.


