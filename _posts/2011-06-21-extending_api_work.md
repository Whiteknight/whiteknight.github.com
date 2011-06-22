---
layout: post
categories: [Parrot, Extending]
title: Extending API Work
---

What is the extending API? We have an Embedding API now. That took a lot of
planning and effort but it provides a very good, consistent interface for
using Parrot from an embedding application. Everything else is for extenders.

So what's the difference?

An embedding application is one where some program creates a Parrot
interpreter and calls it. The `frontend/parrot/main.c` driver program is a
great example of an embedding application. It provides a thin command-line
interface around libparrot using the embedding API. Basically the embedding
API is used to invoke Parrot functionality from a place in code where Parrot's
GC is not configured and operational, where there is not protection from
Parrot exceptions, and there aren't active Parrot `CallContext` around.

An extending situation, on the other hand, is called from within Parrot.
Think about a dynpmc, or a dynoplib, or a library loaded through NCI. In an
extending situation the Parrot GC is configured and sweeps the stack, the
Parrot exceptions system is ready to handle thrown exceptions, and there are
active CallContext PMCs around to maintain a call chain. You can do things in
an extending situation that you can't do in an embedding one: throw
exceptions, create PMC and STRING headers without having to manually anchor
them, access the contents of Parrot registers, examine and interact with the
current CallContext, etc.

As a general, abstract view of the C system stack, you would typically have
this kind of layout:

    Embedding Area  -->  libparrot  -->  Extending Area

The Embedding application calls libparrot, libparrot calls the extending
code. Of course the existance of threads and callbacks complicate this graph
sigificantly, but this is the main relation. If you are writing code and you
can't tell where on this graph it exists, ask and I'll help you figure it out.

Notice that some projects have code that exists in all three of these domains.
It's not exclusive.

We have an nice, well-defined embedding API. There are some incomplete areas
where we could definitely be adding more functionality, but it's pretty good
right now and is serving the needs of many of our users. For the most part,
it's "mission accomplished" on the embedding API. The Extending API is a
different story, however.

The problem is that the extending API isn't well-defined. We don't have a list
of things that are and are not part of the extending API. We don't have
guidelines that say what the API functions look like. We don't have many docs
to tell prospective users how to use the API. We don't know if we even have
an extending API, or if it is and always will be a vague concept. Imagine in
your head that I am waving my hands around to indicate vagueness and
nebulousness.

I don't think it should be vague. I think we need to lay out what our
extending API is and what it is not. I think we need to document how to use
it. It's a huge project, but I think we can tackle it in small bits.

First off, what is the extending API? libparrot, like all dynamically-loaded
libraries exports a list of symbols. Those exported symbols can be called
by other applications and libraries that link to it. Symbols which are not
exported cannot be linked to from another executable. I'm simplifying, so bare
with me. In Parrot, functions marked with the identifier `PARROT_EXPORT` are
exported. So, it seems, those functions form our publically-visible API. The
Embedding API uses `PARROT_API`, which is functionally identical but visually
distinct. So, all exported functions which are marked with PARROT_EXPORT *but
not* PARROT_API are candidates to be part of the extending API. Ideally, I
think the two groups should perfectly overlap. That is, every extending API
function should be marked PARROT_EXPORT, and everything marked PARROT_EXPORT
should be accepted as part of the extending API. The two should be one and the
same. They currently aren't for a variety of historical reasons, but they
should be.

The Embedding API names all its functions "Parrot_api_*". If a function is
named that, it's in the embedding API. If not, it isn't. Simple.

The extending API is a little bit different, because it comprises functions
from a variety of subsystems. Each subsystem has it's own naming convention.
In general, they are all "Parrot_XXX_*", where "XXX" is a 2- or 3-letter
abbreviation for the subsystem. So, "Parrot_gc_*" are API functions for the
GC. "Parrot_oo_*" are functions dealing with object-orientation details.
"Parrot_pmc_*" deal with pmcs. "Parrot_pf_*" deal with packfiles. There are
more, and I won't list them all here. The extending API then is the sum of all
subsystem APIs which are properly named and are marked with PARROT_EXPORT.

That's actually not a bad definition.

There are a few functions in the file `src/extend.c` which are not really part
of any subsystem, and so don't follow that naming convention. There is some
movement to have these functions renamed "Parrot_ext_*", and I think that's a
fine direction to go in. We need to put in the deprecation notices to get them
all renamed, and it might cause a bit of a headache, but overall it does help
to be consistent. I won't say it's absolutely worth it in all cases, but it
will be in most of them.

Recently everybody's favorite dukeleto finished up a grant from the Perl
Foundation to improve test coverage on the extending and embedding functions.
He's done an excellent job on both counts, but the big benefit from his work
may not be the improved test coverage. Instead, he's taken something of an
inventory and a census of these systems, and some of the things he has found
and exposed are quite stinky indeed. For many functions that turn out to be
worthless we do need to jump through the normal deprecation cycle hurdles.
Luckily, many of the functions that I want to kill already have perfectly
usable (or even "superior") alternatives so the transition should not be too
painful.

In some ways working on the extending API could turn out to be a bigger
project than the embedding API was: There isn't a lot of code to be modified,
but there is a huge volume of existing code both in parrot and in various
projects around the ecosystem that will need to be updated. Plus, many of the
changes are function naming changes and by necessity that's going to require
a deprecation cycle (or several) to do correctly.

One of the first steps for working to improve the extending API is to take a
census of sorts: We need to get a list of functions exported by libparrot.
Then we need to go down the list line-by-line to pick out the functions that
don't belong, and find a way to replace each. Many instances will be great
small projects for new hackers to get involved with. Some will be hairy and
difficult enough to keep experienced Parrot hackers busy. The great thing
about it is that we can tackle these one at a time, at our leisure.

Revamping the extending API isn't high on my list of priorities, but it is
something I plan to pick at here and there when I have a spare moment. If
other people are interested in working on it as well, let me know and we can
colaborate.

Speaking of my priorities, I'll be writing about some of those in the coming
days.
