---
layout: post
categories: [Parrot, ParrotProductManagement, Embedding]
title: Parrot's Embedding API
---

Let me take the time to enumerate some tasks that I can think of off the top
of my head that the embedding API is going to need to support, and that the
[embedding API team](/2010/11/05/embedding_api_team.html) are going to have
to implement:

1. Creating, initializing, and destroying the interpreter object. This includes
creating multiple interpreters in a single program, if necessary, and managing
parent/child relationships if needed.
2. Retrieving compiler objects, compiling code files into bytecode, saving
bytecode to file, loading precompiled bytecode files into the interpreter.
Running the programs.
3. Creating, examining, and manipulating individual core Parrot data types.
This includes STRING and PMC objects. For STRINGs we are going to need to
provide access to a large variety of operations. For PMCs, we are going to
need to expose much of the VTABLE interface (or, if changes happen beforehand,
support whatever new interface is created).

Now that we know generally what we want the embedding API to be able to do, we
need to start deciding on the best ways to make that happen. My current vision
is that the embedding API will only really operate on the four core Parrot
data types: PMC, STRING, INTVAL, and FLOATVAL. In the embedding world, these
types are aliased Parrot_PMC, Parrot_String, Parrot_Int, and Parrot_Float, but
I will use the slightly more concise internal names for brevity. I would like
this to be a hard limitation. That is, I don't want to be operating on raw
Parrot_Interp pointers, raw packfile structure pointers, or anything like
that. I want to be returning and using ParrotInterpreter PMCs, PackFile PMCs,
and the like. The raw interp by itself is useless to the embedding user
because all pointers are explicitly treated as being opaque: You can't do
anything safe or sane with a raw Parrot_Interp pointer in an embedding
program. And if you do, we absolutely won't support it. If there is one, this
is a bright spot in our deprecation policy: Pointers and structures are
considered to be opaque.

Of course, if the pointer is opaque and we don't want people to be using it,
why return it to the user in the first place? Give them something that they
can actually use: The ParrotInterpreter PMC. This does create some
difficulties, but it could also create some new opportunities. Overall, I
see it as a win.

One of the biggest areas of apprehension I have going into this project is
to determine what the behavior should be for handling exceptions. Keep in mind
that almost all functions in the embedding API could potentially lead to an
exception being thrown in the interpreter. Current behavior, which is
extremely stupid if you're not looking at
[libparrot as our primary product](/2010/10/21/product_management_team.html)
and the Parrot executable as being a simple embedding application for it, is
for libparrot itself to call the C language `exit(1)` function in the case of
an unhandled exception. This means that every embedding application
*must* either define a fallback exception handler which must remain in scope
for every single Parrot API call, or it must be tolerant of Parrot trying to
exit the program for every unhandled internal error. Doesn't sound like a
great choice to me. I'll show why this is first, then I'll discuss what my
planned solution is.

Let's take a look at a contrived example. We have an embedding application
which defines custom IO behavior by creating a wrapper PMC type around
Parrot's built-in FileHandle type. The embedding application calls into
Parrot, which inevitably calls some kind of `printf` or similar operation. The
IO request calls back into code from the embedding application to be handled.
At this point, our C stack looks like this:

    1) Embedding Parent Code main routine
        |
        V
    2) Parrot Code
        |
        V
    3) Embedding Parent Code IO routine
        |
        V
    4) Parrot FileHandle PMC

At this point, we throw an (unhandled) exception. Pop quiz hot shot: What
happens now?

Well, what currently happens is that Parrot forces the application to exit
abruptly. What *should* happen instead is a little bit more tricky to
determine.

In the basic case, the embedding application has defined an exception handler
of last resort that gets called when no other handlers are found. That
exception is handled gracefully, but because we're at the C level it isn't
going to be possible to resume execution. Longjmp lets you travel up the C
stack, you can never jump back down it. Since we're well below the runloop,
and possibly recursed into a second runloop, there's absolutely no way that we
can consider resuming. From the standpoint of the embedding application, there
is no way to know this either. The embedding application has no idea what the
C stack looked like between itself and the point of the thrown exception.
After handling the exception, the embedding application can attempt to salvage
the Parrot interpreter and execute a new operation, or do something else.

What happens if the embedding application does not define that handler of last
recourse? Parrot shouldn't just call `exit`, that's rude and in some cases
dangerous. On RTEMS, calling `exit` resets the entire operating system! Not
something we want happening on an embedded device in a car, on an airplane, or
shot into space. Of course, Parrot has to o *something*, and execution needs
to go *somewhere*. Without a defined handler, and without being able to call
`exit`, we need something to do. Specifically, we need to exit back to the
embedding application and communicate the error condition so that it can
decide on the next course of action.

The solution that I can see is for Parrot to define it's own last-resort
exception handlers on every API call in. These don't need to be complete
exception handlers as we know them, because they don't actually "handle"
anything. What it does is give Parrot an action to take when all else fails:
jump back to where you came from with some kind of error message or
diagnostics information and let the embedding application deal with the mess.

Since we have Exception PMCs, and other PMCs which can contain result codes
and status information, it makes sense to return a status PMC from every API
call that could potentially throw an exception. This does raise the
semi-predicate problem if we are expecting to return a PMC. Consider this
case:

{% highlight c %}

Parrot_PMC* ex = Parrot_pmc_new(interp, enum_class_Exception);

{% endhighlight %}

When we are expecting to receive an exception PMC as part of normal operation,
how do we differentiate when an error has occured? The answer, as I can see
it, is to always pass status information as the return value, and any other
expected values through pointers in the parameters list. After all, Parrot
has no problems returning multiple values from various functions and methods,
so why should our C API be any more limiting?

Here's an example of all the things I've talked about:

{% highlight c %}

Parrot_PMC * interp = Parrot_api_create_interpreter(NULL);
Parrot_api_initialize_interpreter(interp);
Parrot_PMC *n;
Parrot_PMC* status = Parrot_api_new_pmc(interp, type, &n);

{% endhighlight %}

If `status` is null, we know the API call succeeded. If it was an exception or
some other type of value, we have to examine it to see what the outcome is.

Another option, which may be simpler, is to do something analogous to the
Win32 `GetLastError` function:

{% highlight c %}

Parrot_PMC *n;
BOOL status = Parrot_api_new_pmc(interp, type, &n);
if (!status) {
    PMC *err = Parrot_api_last_error(interp);
    ...
}

{% endhighlight %}

We can think a little bit about how exactly we want to communicate error
conditions, but using the C function return value to do it is the underlying
point I want to communicate.

In closing, here's an idea for what a basic API function definition could be:

{% highlight c %}

PARROT_EMBED_API
INTVAL
Parrot_api_new_pmc(PMC* interp_pmc, INTVAL type, PMC **val) {
    jmp_buf buf;
    Parrot_Interp * const interp = Parrot_get_interp_ptr(interp_pmc);
    if (!setjmp(buf)) {
        Parrot_interp_set_callback_point(interp, buf);
        *val = Parrot_pmc_new(interp, type);
        return 1;
    }
    else {
        Parrot_interp_save_last_error(interp);
        return 0;
    }
}

{% endhighlight %}

Or, through the magic of the C preprocessor:

{% highlight c %}

PARROT_EMBED_API
INTVAL
Parrot_api_new_pmc(PMC* interp_pmc, INTVAL type, PMC **val) {
    API_CALLIN_START(interp_pmc, interp);
    *val = Parrot_pmc_new(interp, type);
    API_CALLIN_END(interp_pmc, interp);
}

{% endhighlight %}
