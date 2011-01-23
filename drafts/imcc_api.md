The work to [clean up IMCC and eventually rip it out of libparrot][imcc_steps]
is progressing nicely. I've been able to cut out a hell of a lot of cruft,
duplicated code, and other oddities so far and am not seeing any reason to
stop yet. With the cleanup moving along so nicely, it's time to start thinking
about the next stage in the process: giving IMCC a nice interface so that we
can wrap it up into it's own library and move it out of libparrot. That's
the point of this post.

We don't want to put a huge amount of effort into the interface for IMCC. It's
not libparrot's [embedding API][embed_api] for instance. However, it is
something that we are going to be using for quite some time and it does make
sense that it not be completely garbage.

To start, I think all IMCC api functions are going to be named "`imcc_`",
similar to how the Parrot embedding API functions are all named
"`Parrot_api_`". IMCC doesn't throw exceptions in the same way that Parrot's
interpreter might, so we don't need to go through the contortions we did
in the embedding API to deal with unhandled exceptions. IMCC does use a very
primitive form of exceptions internally, but those are not exposed to the
user of IMCC.

Let me just start brain-dumping what this interface could be like:

{% highlight c %}

/* Create the imcc_info structure */
imcc_info_t *info = imcc_new(interp);

/* Set an input source. This is either a file or a string of code */
imcc_set_input_file(info, filename, is_pasm);
imcc_set_input_string(info, string, is_pasm);

/* Set some options that may be set on the command-line */
imcc_set_debug_mode(info, debug_flags, yy_debug_flags);
imcc_set_warning_mode(info, warning_mode);
imcc_set_verbosity(info, verbose_mode);
imcc_set_optimization_level(info, 2);

/* Compile the input. Automatically closes the input file, if necessary */
Parrot_PMC packfile = imcc_compile(info);

imcc_destroy(info);

{% endhighlight %}

I think that's a pretty clean and concise API. Notice that there are two
possible functions for setting an input source, and only one function
necessary to run a compilation from it. That's pretty easy to understand,
pretty easy to use, and almost pleasant when you look at it.

The thing that probably represents the largest amount of work here in this
snippet is that first line. Right now, the Parrot interpreter structure
contains an `imcc_info_t` field. I'm going to remove that field and turn the
tables around: the `imcc_info_t` structure is going to contain an interpreter
pointer instead. This way we can conceivably have multiple active IMCC parsers
active for each interp, or maybe none at all.

The next hardest part is going to be the unification of the various interface
functions into a single `imcc_compile` function. This is where I am focusing
some effort now, but I may have to work on a few other areas before it can
be completed.

The least difficult part, though possibly the most tedious, is moving the
remaining command-line parsing logic from IMCC into the Parrot executable
front-end. This involves a lot of copy+paste, and a lot of testing. Move one
thing, build, run all tests. Wash, rinse, repeat. It's not hard but it can be
a little boring. I hope to have this part done within the next few days.

With the API in place, the next step is easy: Create a new PMC type to
wrap this interface. I'll post progress as it happens.
