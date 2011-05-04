I've been kicking around an idea for how to improve the Parrot Calling
Conventions (PCC). Two years ago we went through the system and did a major
refactor on that system, unifying all the various code paths to use a single
central dispatch and argument processing mechanism. The new system uses the
CallContext PMC type to aggregate all parameters for a call. The caller always
puts arguments into the CallContext, and the callee always retrieves
parameters from it. Since Parrot uses CPS, the same mechanism is used for
returns as well.

This is a great system, and I don't want to fundamentally change the way it
works. What I do want to do is change the way we access parameters on the
callee side. First I'm going to explain how things currently work, and then
I'm going to talk about what my ideas are for changing it.

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

I have two problems with this system: The first are the signatures, which are
really weird and suboptimal ways to store the information about the function
arguments and parameters. The second problem is with `fill_params`, which
needlessly loops over the signature. My problem with both of these two things
is this: We have all the information we need at compile time, including
information about how many args/params we have, what order they are in, what
flags and special conditions they have, etc.
