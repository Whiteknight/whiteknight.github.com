---
layout: post
title: Embedding API Initialization Details
categories: [Parrot, Embedding, ParrotProductManagement]
---

My last few posts about initialization have attracted a good amount of
attention and feedback, which I am grateful for. I've started moving on to the
next stage of trying to get my new prototype API to build and run. So far this
is turning out to be the most difficult stage of the process, and I'll explain
why.

When you write up a large amount of code and try to build it the first time,
you invariably wind up with a few problems. There are syntax errors and all
sorts of other problems detected at compilation. This is to be expected and
really isn't eating up too much of my time or effort. The far harder part is
getting everything fitting together in a reasonable way, trying to get all
the type and value definitions available in the correct places, and making
sure all the dependencies are in order is a much harder problem. In most cases
it's time consuming without requiring too much mental effort. In other cases,
the mental effort required to make things work the way they should is not
trivial.

In an earlier post I said that I only wanted to expose the [primary four data
types][datatypespost] to the embedding API. The majority of the API should
operate only on `Parrot_PMC`, `Parrot_String`, `Parrot_Int`, and `Parrot_Num`
types. This helps to shield the user from the added complexity of the dozens
of types Parrot uses internally, helps encapsulate Parrot by not exposing
raw pointers which are supposed to be treated as opaque, and really helps to
simplify things on all fronts. We need fewer conversion methods if we only
have 4 types of data to work with, and we need fewer support routines to
manage them. In short, I think it's a good idea and I've been trying to stick
to it as I write the new code.

[datatypespost]:/2010/11/06/embedding_api.html

The problem comes in a few edge cases, especially during [interpreter
initialization][interpinitialization]. The fact is that there are several flag
types that Parrot is
relying on for even the most basic initializations. What kind of runcore to
use? That's an enum. What GC to use? That's an enum. What warnings to enable?
Enum. Debugging options? Enum. Tracing capabilities? Enum. All of these
represent custom C types that need to be specified and *this isn't even the
complete list* of types needed to initialize.

[interpinitialization]: /2010/11/13/embed_api_comment_response.html

The GC selector enum is defined in the private header file for the GC system,
so `src/main.c` was having to include `gc/gc_private.h`. Excuse me while I
cough up some blood.

Here's the list of header files that `src/main.c` was including before the API
changes:

    #include "parrot/parrot.h"
    #include "parrot/embed.h"
    #include "parrot/imcc.h"
    #include "parrot/longopt.h"
    #include "parrot/runcore_api.h"
    #include "pmc/pmc_callcontext.h"
    #include "gc/gc_private.h"

That's a pretty ugly list. Here's where we are right now in my fork:

    #include "parrot/api.h"
    #include "parrot/imcc.h"
    #include "parrot/longopt.h"

This is a pretty good improvement, and is better when you consider that in
the fork longopt and imcc will not be built in with libparrot. The only
header file required for most embedding applications will be `parrot/api.h`,
and I don't think that's too steep a requirement.

The problem, of course, is that all those other header files provided a whole
slew of flag, macro and type definitions that `src/main.c` was relying on.
How do we deal with that?

First it will be worthwhile for me to define a little bit of terminology that
I will use for the rest of this post. "private" symbols are those that are
visible inside libparrot, but are not visible outside it. "public" symbols are
the opposite: intended for use by embedding applications but not visible or
usable from inside libparrot. "public+private" symbols are those that are
visible everywhere.

The problem I am having is this: The new include file `parrot/api.h` is
*public*. It is intended for use by embedding applications and is not
intended to be used within libparrot itself. The master include
`parrot/parrot.h` is becoming *private*. Embedders won't see it or use it at
all. However, some things like warning flags really want to be
*public+private*, or at least be private with some way to manipulate them
externally. I have a few paths to consider:

1. We could replace these flags with `char *` strings, and parse them out to
   get the real values. I've already done this with the GC flags,
   though I'm not sure I want to do this with some of the more common
   flag-setting options.
2. I could duplicate the enum/flag definitions into a new header file. This
   would solve my problem the most quickly, but it would heavily violate the
   principle of [DRY][] and it creates a huge maintenance burden trying to keep
   two differen definitions in sync.
3. I could take the time to tease the headerfiles apart to include all
   public/private flags in a handful of files
4. Forget flags and values. Instead, we have one API function to set or clear
   each and every flag for each and every option.

[DRY]: http://en.wikipedia.org/wiki/DRY

I think we all can agree that option 2 above is not desirable and option 4
has the same problem to a larger magnitude. Option 1 makes sense
in cases like setting the GC core where it only needs to be set once, is set
early on, and has a sane default. The winner then appears to be option 3.

My plan is to create a new include file, `parrot/flags.h` to include the
public+private enum definitions. I'll include that from `parrot/api.h` so that
the user has access to all the necessary features in a concise and natural
way.

The immediate tasks for getting this work done are as follows:

1. Sort out all these flags and enums. This is necessary just to get things
to build.
2. Separate `longopt` routines so that they are not included in libparrot
3. Fill in some of the "TODO" blanks in the various API functions.
4. Test and debug problems

I think we can get this nailed down this week or next. When we get to that
point I'm going to push this work to a branch in the primary Parrot repository
to start getting more hands and eyes on it.

