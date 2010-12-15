---
layout: post
categories: [Parrot, Packfiles]
title: Packfiles, first steps
---

Every time I start thinking about a project that needs to get done in Parrot,
and I start tracing out the requirements and the underlying problems, I almost
always am lead to a single root problem: Packfiles. "Packfile" is the term we
use internally to talk about Parrot's bytecode files and their format. In
Parrot packfiles are typically an internal-only structure, but within the last
year or so they have gotten a PMC interface which makes them accessible and
partially usable from PIR code running on Parrot.

One thing that I think we need to do in Parrot is to make Packfiles work
better and to use a proper encapsulating API. This task, like many other
projects to refactor subsystems, can be broken up into two phases of primary
development:

1. Encapsulate the system behind a proper API
2. Change the internals in such a way as to keep the API working the same

Both of these tasks can be extremely difficult, and affect each other in
complicated ways. If we create a Packfile API but it turns out to be extremely
lousy, then when we go to change the internals we may find that we cannot
satisfy it's requirements. Or we may find that the users of the API hate it,
and we will be forced to change the API and much of the code behind it. We
want to make sure we take some time to properly design the system and consider
not only the current uses of it but also the kinds of uses we want to support
in the future. Right now we are entering this crucial design phase for
Packfiles.

I want to talk a little bit about what Packfiles currently are, and then move
on to talk about what I think a new API for them should look like.

Packfiles are one of the oldest, largest, and messiest subsystems in Parrot.
IMCC, the PIR frontend that I love to hate, is the only real producer of
functional packfiles, so those twp systems have become inextricably linked.
IMCC calls private-looking functions in the Packfile systems and directly
uses internal structures and pointers from it. The Packfile system needs to
frequently call utility functions in IMCC to perform even some of the most
basic tasks. In short, the two are linked, and if we want to get rid of IMCC
ever we are going to need a serious development effort in the packfile code
to cut it free.

Packfiles are represented by a series of C structures. Packfile,
Packfile_Header, Packfile_Directory, Packfile_Segment, etc. There are about
a dozen or more structures like this. I am going to be honest and admit that
I really have only a vague idea about how they all fit together. This is an
area where I need to sit down and just start reading code, no matter how
painful an experience it turns out to be.

In addition to these structures there is a series of PackFile PMC types with
similar names. Each PackFile PMC type interacts with one or more of these
C structure types in a general way. Right now the PMC types don't provide a
complete wrapper for the Packfile structures, but they do provide enough of
an interface to do basic testing of Packfiles and to be able to create
extremely small toy packfiles from PIR. bacek and cotto have done some work
on little examples of this mechanism. I don't remember exactly where they are,
but I do distinctly remember that the examples were complicated and long
compared to the bytecode it produced.

Actually, right now the PackFile PMCs don't really work at all. One of the
first things that needs to get done, and I am going to try to help with this,
is to get those PMC types working correctly in current Parrot. This may be a
very big or very small task, depending on how much code has moved since they
were last functioning.

Once those PMCs are working properly again, we can implement a bunch of tests
to keep them working and then talk about what we want to do next.

What I think we need to do is make the PackFile PMCs a more faithful and
feature-full wrapper around the Packfile structure types. Once we have this,
we can start to create an API encapsulation boundary that only passes these
PMC types. Outside the encapsulation boundary we have code that uses the PMCs
and the public interfaces on them only, without having any intimate knowledge
of exactly how they are structured internally. Inside the encapsulation
boundary we can extract pointers to the raw packfile structures and use those
natively; keeping the performance of low-level structure access and remaining
absolutely unaware of what the external API looks like. In the basic sense,
we have two C-level functions:

    PMC * Parrot_pf_get_packfile_pmc(PARROT_INTERP, struct Packfile *pf);
    struct Packfile * Parrot_pf_get_packfile_pointer(PARROT_INTERP, PMC *pfpmc);

One takes a packfile structure pointer and returns the associated PackFile
PMC. The other takes a PackFile PMC and returns the Packfile structure pointer
from within it. These don't even need to be functions, for added performance
they could easily become macros. What's important that is that the packfile
subsystem works on native structure types and everybody else can pass PMCs
around.

If this approach sounds familiar, it's what I did with the [embedding API
work][embedding_pmcs]. Where possible, the new embedding API uses PMCs instead
of lower-level types. Once inside the API boundary we can break out the
lower-level structures and values that we need to work on, but externally we
want to play with PMCs exclusively.

The reality is that everybody in the world doesn't want to be playing with
raw structures and pointers and other garbage. Everybody else wants to be
using a clean and stable interface: an established API and PMC VTABLEs and
METHODs. The nitty-gritty details are too much work to be working with for
most consumers, and this includes most parts of Parrot itself.

Packfiles really represent a core functionality of Parrot. The whole VM relies
on their use to get anything done. Without packfiles, there is nothing for
Parrot to execute. However this central pillar of sorts is in need of some
serious love. Not only do we need to fix the broken bits and clean up the
implementation, but we are actually going to want to build up on top of those
foundations to add some great new features and capabilities to Parrot. We
need, for instance, the ability to create packfiles directly from PIR code,
which means we need an easy PMC-based interface to them. Once we have this
feature in place and working reliably we can be creating compiler libraries
on top of Parrot, and start having HLLs output PBC directly instead of
outputting PIR which is then compiled to PBC by IMCC. Combine these fun new
powers with the added flexibility of the new embedding API and I think we are
going to start to see some novel and amazing new tools using Parrot and
running on top of it. The first step in all this is getting Packfiles fixed
and cleaned up, which I think we are going to do soon.

[embedding_pmcs]:
