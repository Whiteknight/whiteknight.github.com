I've been kicking around an idea for how to improve the Parrot Calling
Conventions (PCC). Two years ago we went through the system and did a major
refactor on that system, unifying all the various code paths to use a single
central dispatch and argument processing mechanism. The new system uses the
CallContext PMC type to aggregate all parameters for a call. The caller always
puts arguments into the CallContext, and the callee always retrieves
parameters from it. Since Parrot uses CPS, the same mechanism is used for
returns as well.

This is a great system, and I don't want to fundamentally change the way it
works. Back when we did the refactors the hope was that we would be able to
find new opportunities for optimization and improvement, and we really haven't
delivered on this the way we should have. I think there are plenty of things
we can do to improve performance, simplify some of the hot codepaths, and make
better use of compile-time information which we currently discard. First
I'm going to explain how things currently work, and then I'm going to talk
about what my ideas are for changing it. This is not an exhaustive list of
possible improvements, just the few that I have in mind right now.

In Parrot, invocations (function, method, exception handler, and continuation)
are a multi-step process. First, we have to lookup the subroutine object. This
can be either a constant value, a Sub stored in a NameSpace, or a method
stored in an object/class. We create a CallContext object by passing in a
signature array with a series of flags and using those flags to insert a list
of arguments into the CallContext. We store the CallContext in the
interpreter, then invoke the Sub object.

On the callee side, we use a signature array along with the CallContext to
unpack the arguments into registers. The unpacking happenings in a function
called `fill_params`, which is a huge behemoth of a function. Actually that's
not fair. It's a behemoth in the context of Parrot, where we tend to
appreciate shorter functions. Other projects might not think it's so large in
comparison. `fill_params` uses a series of loops to get the arguments. It
loops over the flags in the signature, and depending on what each flag is it
pulls the next value out of the CallContext, and inserts it into the specified
register.

I have two problems with this system: The first are the signatures, integer
array PMCs stored as constants in the packfile. These are really weird and
suboptimal ways to store the information about the function arguments and
parameters. What is obnoxious about these is that we have to loop over these
signature arrays to pull arguments out of the caller context. However, at
compilation time we already know what arguments to pass, and could easily
generate a custom sequence of argument passing args without ever needing to
use a signature. If we need a signature, such as in multi-dispatch, we can
generate those lazily on the fly. The second problem is with `fill_params`,
which needlessly loops over the parameter signature to extract parameters and
insert them into the callee context. I dislike this for exactly the same
reason, we have the information we need at PIR compile time and we can easily
generate a stream of custom fetch ops for the specific things we need instead
of generating a signature at compile time and then needing to interpret that
signature at runtime every time we make a call. In short, we have plenty of
information available at compile time, but we discard it and compress it into
signatures. Every time we make a call at runtime, *every time*, we need to
loop over two signatures: one to fetch args from the caller, and one to
deposit params into the callee. Multiply this by two when you remember that
Parrot uses CPS, so every return is also a call (along with two more
signatures). I suggest we shouldn't need *any* signature arrays for most
calls, and I suggest we shouldn't need *any* loops for most calls either.

When we make a call in PIR, it typically looks like this:

    (ret1, ret2, ...) = 'function'(arg1, arg2, ...)

This, as our developers should be well aware, is syntactic sugar over a
sequence of lower-level ops. Here's a basic example of what the generated
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
to get fixed if we want concurrency, but won't necessarily bring any
performance improvements.

The next thing to see is that the argument-related ops, `set_args`,
`get_params`, `set_returns` and `get_results` are all variadic. In fact, they
are the only variadic ops in the entire system and there is more than a little
bit of logic throughout Parrot core that could be simplified if we had no
variadic ops at all. What we do is this: The signature tells the number and
the types of the various arguments. Each argument is a pointer stored directly
in the bytecode stream. When we loop over the signature, we dereference
subsequent locations in the bytecode stream to fetch the values, and push
them into the CallContext from there. Life will get a little bit easier for
people working on compilers and related tools (disassemblers, code analyzers,
etc) if we don't have these kinds of ops. It isn't a concern for most users,
but then again having better tools can help those people too.

The set opcodes, `set_args` and `set_returns` loop over the signature and
push values from the caller context into the CallContext PMC for the call.
This process is relatively straight-forward, although not without some
inefficiencies. On the fetch side, `get_params` and `get_results`, need to
do much harder work. This is where that `fill_params` function comes in.
`fill_params` loops over the receiver signature, and starts hunting for
values in the CallContext to match the specific needs. `fill_params` needs
to compare the number of passed arguments with the number of expected
parameters and, depending on settings, throw errors. It needs to determine
if `:optional` params get filled or not. It needs to handle `:slurpy`. It
needs to fetch named arguments by args, determining what to do if we don't
have named arguments that we've asked for and if we have named arguments that
we have not asked for. As if that wasn't complicated enough, we need to
correctly handle `:named :optional` and other combinations. Also,
`fill_params` needs to coerce arguments to the correct types. For instance,
if we pass an integer and expect a PMC, we need to autobox. This is a heck of
a lot of work for this single function, and the results performance-wise are
not exemplary.

So what can we do? What should this look like? How can we cut out cruft?

I suggest we should get away from the four do-it-all ops, `set_args`,
`get_params`, `set_returns` and `get_results`. They are all problematic and
they all need to go. Second, I suggest that we have all the information we
need to make an efficient call at compile time. We shouldn't condense the
details down into a signature, then have to go to lengths to interpret that
signature in a large loop. CallContext is an object with array-like and
hash-like interfaces, and we have opcodes for interacting with arrays and
hashes. Since the opcodes call the corresponding vtables directly, and those
same vtables are used in `fill_params` and elsewhere to put values into and
extract values from the CallContext, there's no speed savings doing it all
internally. The only things we would need are a handful of new ops to handle
some of the special cases like `:optional`, `:slurpy` and `:flat`.

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

Notice that not only should this sequence be thread-safe, but that it's simple
and straight forward. This sequence would be easy to disassemble and analyze,
requires no signature constants from the packfile, and requires no loops. We
know exactly what arguments we want to pass, and we just pass them. We know
exactly what results we want to fetch, so we just fetch them. No muss, no
fuss.

Notice also that we create the CallContext and pass it around in user code.
We don't manage it and store it internally, which means the compiler can have
opportunities to recycle and reuse them between similar calls in a loop, for
instance. In the current system, there really is no opportunity to share
CallContext PMCs between subsequent calls, so we need to create them fresh
for each one. That's a lot of churn and a lot of GC pressure for no benefit.

How do we get there?

The hard part is getting there. Right now we would need to modify IMCC, which
is no small task. I tried making these changes in a branch as proof-of-concept
and the results were not pretty. It's not impossible, but I still have a lot
to learn about IMCC before I get deeper into this project. Considering the
amount of effort required, it's hard to do it for a quick throwaway prototype.
On the other hand, these kinds of changes should be much easier to make in
PIRATE, and that's where I might start doing some prototyping.

So long as everybody uses the PIR sugar and does not attempt to use the PASM
opcodes directly, the switchover should be mostly transparent from the
perspective of the user.
we can get rid of variadic ops,


