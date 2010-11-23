---
layout: post
title: Embedding API Branch Builds, Mostly
categories: [Parrot, Embedding]
---

As of tonight, the parrot executable in the `embed_api` branch builds and can
be used to execute small programs. I say "small" because there is still one
crucial detail missing: library search paths.

The configuration stage of the Parrot build populates a huge (some would say
"too huge") hash of information gathered. Everything from the compiler and
compiler commandline flags used to build parrot, environment settings,
the platform and the platform's various capabilities, etc. All of these things
are stored in the configuration hash. During the build, this hash is
serialized and compiled in to the Parrot executable. It's made available as
the `IGLOBALS_CONFIG_HASH` entry in the interpreter's array of globals, and
can be accessed from various places. Several extension projects, [PLA][]
included, use this information to direct and inform their builds to create
compatible binaries.

[PLA]: http://whiteknight.github.com/parrot-linear-algebra

There has always been a pretty big problem with the configuration hash,
however. The problem is the way it is included in the Parrot executable. A
utility called `parrot_config_c.pir` writes out the raw bytes of the
serialized config hash into a C source code file. Here's a short example:

{% highlight c %}

static const unsigned char parrot_config[] = {
    0xfe, 0x50, 0x42, 0x43, 0x0d, 0x0a, 0x1a, 0x0a,
    0x08, 0x00, 0x00, 0x02, 0x09, 0x01, 0x09, 0x01,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x04, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    ...

{% endhighlight %}

...And it continues on like that for several thousand lines of extremely
uninteresting reading. That static variable gets passed into libparrot
*before* the creation of the first interpreter. Thereafter, whenever an
interpreter is created, the creation routine reads this serialized byte
stream and deserializes it back into a local copy of the config hash. The
values in that config hash are used for, among other things, initializing the
lists of library search paths.

Parrot's build process has two stages. The first is called miniparrot.
miniparrot is a copy of the parrot executable except it has an empty config
hash compiled in to it. Miniparrot is used to serialize the configuration
hash, which then moves to the next step to build the actual parrot executable
with the actual configuration details included.

The embedding API, as it stands now, has broken that initialization up into
two parts. In the first part, we set up a default list of search paths for a
Parrot that doesn't have a configuration hash. Then, if we set a configuration
hash later, we try to update the search paths to include information from the
hash. I haven't finished the implementation of this last part, but in my mind
it's already done and it is completely awesome.

Another nice benefit of the new API is that instead of passing in an array of
bytes, we can now pass in a fully-formed PMC. And since we're using PMCs, it
can all be done after the interpreter has been created and the GC has been
initialized. Instead of needing to be compiled in, we can load or dynamically
construct the config hash any way the embedding application wants to. That's
if the embedding application wants to load it at all. In short, the new
version is going to be much nicer than we have now. At least, it will be as
soon as it's complete.

I'm hoping to get the details of this new mechanism sorted out by tomorrow
evening or Wednesday at the latest. At that point I think we can open up the
discussion about the new API to the wider community, and start working to
get the branch mergable.

We're in the home stretch now. I'm very excited about it.
