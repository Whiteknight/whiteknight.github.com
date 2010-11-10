---
layout: post
title: Embedding API Sneak Preview
categories: [Parrot, Embedding]
---

[Parrot is now on Git][parrotongit], and I've already created one of the first
public forks of the repo. This fork is where I am planning, at least initially
to do the actual work of implementing the new embedding API.

Why a fork, instead of a branch? We can get into the details more in a later
post, but the short version is that I wanted to play with it. Since Parrot has
just moved to git and we as a community have not settled onto a set of best
practices on how to use it, we do need to do some experimentation about how to
best do things. So far as the master repository is concerned, it should
basically look like apples and apples anyway.

Getting onto my main topic of this post, I've been doing a little bit of work
on the embedding API in my new fork. I'm in the process of slowly translating
the Parrot executable code to use the new API, and am making decent progress
at it. What I want to do here is show a quick preview of what the new code
is looking like, and include some short explanations about why things look the
way they do and maybe offer up some ideas for alternatives. Feedback, as
always, is highly encouraged.

{% highlight c %}

Parrot_PMC * interp;
Parrot_Interp * raw_interp = Parrot_api_make_interpreter(NULL, PARROT_NO_FLAGS);

/* Select GC, and other pre-initialization options here */

if (!(Parrot_api_initialize_interpreter(raw_interp, NULL, &interp) &&
      Parrot_api_set_runcore(interp, runcore, trace) &&
      Parrot_api_set_executable_name(interp, Parrot_str_new(argv[0])))) {
    fprintf(stderr, "PARROT VM: Could not initialize interpreter: %s",
        Parrot_api_last_error_message(interp));
    exit(EXIT_FAILURE);
}

/* This part hasn't been translated yet */

int status = imcc_run(raw_interp, ...);
if (status)
    imcc_run_pbc(raw_interp, ...);

if (!(Parrot_api_destroy(interp) &&
      Parrot_api_exit(interp, 0))) {
    fprintf(stderr, "PARROT VM: Error during interp destruction: %s",
        Parrot_api_last_error_message(interp));
    exit(EXIT_FAILURE);
}
exit(EXIT_SUCCESS);

{% endhighlight %}

There are a few things that are really worth pointing out now because I'm just
not sure what exactly to do with them. As I mentioned in my earlier post about
the [design of the embedding API][apidesign] I wanted all interactions with
Parrot Interpreter to be done through the ParrotInterpreter PMC type. I still
stand by this earlier assertion, but I run into a problem at the very first
function call sequence in the Parrot executable. The problem is that we need
to set information about which GC subsystem to use, information which can
currently be set on the commandline. In order to set the GC system, we need
to have a `Parrot_Interp` object handy, but we cannot have created a `PMC*`
yet. It's the GC that allocates PMCs, after all. What we have is a chicken and
egg problem, and in this case the chicken must come first.

One solution is to do this two-stage initialization like I have in the example
above: Create the raw interp, use it to set the lowest-level startup options,
and then initialize to wrap it all up into a PMC for all other uses. This
seems fine for now, but a little bit problematic. After all, it can be very
confusing if the documentation isn't up-to-snuff whether each individual API
call uses the raw interpreter pointer or the interpreter PMC. And then some
APIs won't be callable at certain times or in certain situations, and you run
into a case where we mix up the relationships between raw interpreter pointers
and the corresponding PMCs, etc. In short, it's a big mess and I don't like
it.

What I think I am going to do instead is modify the
`Parrot_api_make_interpreter` function to take some of the low-level options.
Of course, if we have it take a list of options, and the number of those
options increases, we're going to be making many backwards-incompatible
changes to this function. That's unacceptable.

What I think we can do instead is have a structure of initialization options.
The user allocates this initialization structure, fills in any fields that
matter (all are optional), and passes that to `Parrot_api_make_interpreter`
to configure the low-level options and return a properly-wrapped PMC object.

A benefit here is that so long as we only add fields to the end of this
new structure, we can avoid deprecation cycles. Hell, if we pass in a size
argument too, we can completely avoid recompilation of the host application
when libparrot is upgraded.

Here's an example of this idea:

{% highlight c %}

Parrot_PMC * interp;
const Parrot_Int args_size = sizeof(Parrot_Init_Args);
Parrot_Init_Args *args = (Parrot_Init_Args*)calloc(args_size);
args->interp_flags = PARROT_NO_FLAGS;
args->gc_system = PARROT_GC_MS;
args->gc_threshold = 45;
args->hash_seed = 0xdeadbeef;
Parrot_api_make_interpreter(args_size, args, &interp);

{% endhighlight %}

This seems a little bit more messy and I'm not entirely happy with it, but it
does have the benefit that it works well with the deprecation policy and it
prevents us from exposing raw interpreter pointers to the embedding
application. It's definitely a trade-off, and I really want to hear what
embedding application developers have to say about it before I do any more
work on that part of the API.

Another section I modified below is the new `Parrot_api_exit` function. This
function acts as a thin wrapper around the old `Parrot_exit` function. The
difference is that it sets an api jump buffer, and `Parrot_exit` no longer
calls `exit()` immediately. Instead, `Parrot_exit` jumps back to the jump
point if it is set and only calls `exit()` if there is no jump point to go to.
This should preserve existing behavior for anybody who is still calling
`Parrot_exit` directly, but provides new projections for people who want to do
things the *correct* way.

We want this protection wrapper around the `Parrot_exit` function because it
can execute an arbitrary list of exit handler functions, some of which
might be written in PIR and could throw unhandled exceptions. Or, worse than
exceptions, we could get into some huge panic when dismantleing the GC or
any other subsystem when Parrot normally would have no recourse but a speedy
exit. Now, we can catch and handle those situations in the embedding
application more gracefully.

The last section that I really want to talk about is the part that calls
IMCC. The `imcc_run` function passes the command-line arguments to IMCC for
processing, and if any `.pir` or `.pasm` files are specified they are
compiled into PBC. The `imcc_run_pbc` function executes that PBC, if required.
The problem I am having is the same one I mentioned in
[another recent post][imccproblems]; I don't know whether IMCC should be part
of libparrot, or whether it should be external. If external, this is the
call sequence we will likely need to make (without any error handling, for
brevity):

{% highlight c %}

Parrot_PMC * packfile;
Parrot_PMC * compiler;
Parrot_api_register_compiler(interp, "PIR", imcc_compile_pir)
Parrot_api_register_compiler(interp, "PASM", imcc_compile_pasm);
if (STREQ(file_extension, ".pir")) {
    Parrot_api_get_compiler(interp, "PIR", &compiler);
    Parrot_api_compile_file(interp, compiler, filename, &packfile);
} else if (STREQ(file_extension, ".pasm")) {
    Parrot_api_get_compiler(interp, "PASM", &compiler);
    Parrot_api_compile_file(interp, compiler, filename, &packfile);
} else if (STREQ(file_extension, ".pbc"));
    Parrot_api_load_bytecode_file(interp, filename, &packfile);
Parrot_api_execute_packfile(interp, packfile);

{% endhighlight %}

This is a little bit verbose, but highly flexible. The values
`imcc_compile_pir` and `imcc_compile_pasm` above are hypothetical new
function pointers that IMCC would provide to compile a file and return a
PackFile PMC. These don't exist, but should be pretty easy to create.

This last snippet is going to take some work to implement, if this is the
direction we want to go in the long run. The problem, as I mentioned
previously, is that IMCC does a lot of command-line argument processing that
we need to avoid. Instead, the executable should process all the arguments and
make a series of API calls to IMCC to set options. Then, once the options are
set, we pass in the file name and return a PackFile PMC.

In fact, we could load any compiler front-end we want, so long as the compiler
implemented a simple callback with this format, or something similar:

{% highlight c %}
Parrot_Int (*)(PMC *interp, STRING *filename, PMC **packfile)
{% endhighlight %}

So that's my quick sneak preview of the new embedding API. There are a few
questions left unanswered and a lot more work that needs to be done, but
the end product should be very nice and very usable for a wide variety of
tasks. Like I've said several times, *feedback is highly encouraged*


[parrotongit]: http://github.com/parrot/parrot
[apidesign]: /2010/11/06/embedding_api.html
[imccproblems]: /2010/11/09/embedding_problems.html
