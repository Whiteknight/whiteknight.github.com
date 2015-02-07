---
layout: post
title: Embedding API, Unresolved Problems
categories: [Parrot, ParrotProductManagement, Embedding]
---

I've been going down the list of functions I
[mentioned yesterday][api first wave] and slowly prototyping some new API
functions to replace the ones used by the Parrot executable. However, I'm
quickly running into the conundrum of what to do about command-line arguments.

[api first wave]: /2010/11/07/embedding_api_first_wave.html

Commandline arguments are processed in several places throughout Parrot.
Most argument processing happens in the Parrot executables `main.c` and in
IMCC. Also, the definitions of the arguments themselves is defined in yet
another file, `longopt.c`.

`longopt` is a routine that helps with automated argument parsing. It takes an
array of argument definitions and iterates through the `argv` array, returning
options one at a time. I like longopt, and I feel like the core mechanism
should be included in libparrot so embedding applications can make easy use of
it. However, specific argument definitions should not be part of libparrot,
they should be owned by the embedding application. A different front-end
will define different command-line arguments, after all, not all of which
would need to be passed to the embedded Parrot interpreter.

For that matter, passing commandline arguments to an embedded interpreter for
any kind of parsing really seems wrong. There should be API functions
available to set all the different options from anywhere, without making any
assumptions about where those settings will be coming from or what the
command-line interface should be doing. By convention, the `:main` sub in PIR
is called with an array of strings that are received on the commandline, but
that convention has more to do with the behavior of the Parrot executable than
it has anything to do with the underlying interpreter or the PIR language. An
embedding application should be able to specify any combination of arguments
to the `:main` function. We need to expose this kind of flexibility to our
embedders, but we also need to provide convenience methods to convert an
`char **argv` array into a `ResizableStringArray` too.

The next question is IMCC, Parrot's current PIR compiler. IMCC previously
handled all command-line argument parsing, but some of the processing was
moved into the Parrot executable because there were some options added that
needed to be set before interpreter initialization. The lion's share of it is
still in IMCC, however, and that raises some serious problems.

Let me ask an architectural question: Is IMCC part of libparrot, or is it part
of the Parrot executable? Or, I suppose as a third option, is it supposed to
be a separate loadable library? And if it's a separate library, is it loaded
by libparrot or the Parrot executable?

These questions really dig into the heart of what libparrot is. In the long
term, we know that [PIR is going to be replaced by Lorito][pir], which means
that IMCC is going to be removed anyway. That
[could be months away][timeline], and we need answers for the embedding API
*now*.

[pir]: http://trac.parrot.org/parrot/wiki/LoritoRoadmap
[timeline]: /2010/11/04/parrot_of_the_future.html

My personal opinion is that IMCC should be external to libparrot. This
preference creates a few implications: IMCC can continue to process CLI
arguments, but now libparrot cannot assume that IMCC is always loaded, since
embedding applications may not always include it. If you have an HLL
compiler that parses it's own language and outputs Parrot bytecode directly,
there's no reason whatsoever to include IMCC in your application. If, on the
other hand your application may occasionaly require runtime evaluations of
PIR snippets, or loads bytecode libraries that do, you'll want IMCC around
(or, whatever replaces IMCC in the future).

If we have a consistent mechanism by which we can load external language
compilers, this really becomes a non-issue. The embedding application can do
something like this, written in PIR for brevity:

    load_language "PIR"
    $P0 = compreg "PIR"
    $P1 = $P0.'compile'($S0)
    $I0 = $P1()
    return($I0)

And this will go through and load the correct compiler library by name,
without needing to be concerned about whether that compiler library is a
native library or a PBC library.

In the embedding API, we can have a similar sequence:

{% highlight c %}

Parrot_PMC * compiler;
Parrot_PMC * bytecode;
Parrot_Int exitcode;
if (Parrot_api_load_language(interp, "PIR") &&
    Parrot_api_get_compiler(interp, "PIR", &compiler) &&
    Parrot_api_compile_file(interp, compiler, filename, &bytecode) &&
    Parrot_api_load_bytecode(interp, bytecode) &&
    Parrot_api_run(interp, &exitcode))
{
    return exitcode;
}
else
{
    const char * err = Parrot_api_get_last_error(interp);
    fprintf(stderr, err);
    return 1;
}

{% endhighlight %}

That's just one imagining of what the API will look like, there are a few
places where it could be a little less verbose, and few calls in there that
probably will be more verbose. I suspect the equivalent of
`Parrot_api_get_last_error` will probably return a Parrot_STRING instead of a
`const char *`, but who knows what the users want at this point. I don't even
know if this is how we're going to be communicating errors. I strongly suspect
it, but we don't know for certain.

Notice also that if we can't find the IMCC binary just using the search string
`"PIR"`, we can probably specify a search path:

{% highlight c %}
    Parrot_api_load_language(interp, "PIR", "/path/to/imcc");
{% endhighlight %}

That should give maximum flexibility to other embedding applications to load
their own custom compiler frontends, including supplying a custom variant of
an existing compiler. If you're running a precompiled bytecode file you can
probably save a few cycles by not loading the compiler at all, though you
won't be able to do runtime eval on strings without one.

Anyway, let me get back to my original topic: Commandline arguments. No matter
where IMCC lives, or how it is loaded, I don't think that IMCC is the correct
place to be doing any CLI argument processing. First step is to simplify
argument processing in IMCC and reduce duplicate functionality. Second step is
to refactor out all the argument processing code from IMCC, replacing it with
a series of function calls that the embedding application can make if
necessary. The final step, I suppose, is to prevent IMCC from poking into
the Parrot interpreter directly. The last step may be out of the purview of
the embedding API project, since a solid case can be made that a compiler
front-end would use a separate "extending" API instead of the "embedding" API
I am working on now.

Once argument processing logic is removed from IMCC, we can handle all of the
necessary functionality through proper API calls, and be a little bit more
agnostic about using different frontends in the future. Eventually, when we
do decide to kick IMCC out the door for good, it won't be nearly as hard as
trying to do everything at once.

With this idea in place, tomorrow I will give a sneak preview of what the
new embedding API might be looking like when it's complete.
