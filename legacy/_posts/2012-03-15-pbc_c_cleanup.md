---
layout: post
categories: [Parrot, IMCC]
title: Packfile Write API
---

Last night benabik told me about a problem that, while serious, hardly caused
me to raise an eyebrow. Some innocuous-looking code, written while trying to
follow along with some ill-written documentation, lead IMCC to enter into an
infinite loop. I wasn't surprised that IMCC contained such a bug, though I was
surprised that it was so easy to reproduce and the functionality in question
wasn't really tested at all. The code that caused this bug is this:

    .const 'FixedIntegerArray' foo = 'test'

The Sub 'test' was an `:immediate` subroutine which generated a
FixedIntegerArray. For people not familiar with IMCC's workings an
`:immediate` Sub is executed as soon as the packfile is compiled and its entry
in the packfile is replaced with the return value of the Sub. So, the Sub
doesn't exist in the packfile, only the thing that the Sub generated. This is
the mechanism by which arbitrary PMCs can (in theory) be serialized into the
packfile at runtime.

Compare with this syntax:

    .const 'Sub' foo = 'test'

In this example, the local variable 'foo' will refer to the item in the
constants table where the Sub 'test' is (or would be, if it weren't an
`:immediate`). The tag 'Sub' there is a bit of a misnomer, since the PMC in
that slot might not be a Sub. This syntax is basically a really lousy way of
saying "give me the PMC in the constant table in the slot where the given named
'Sub' is or would be." Also, that type information isn't really used
for anything, since PIR is a dynamic language. At first glance you might
suspect that the `.const` directive can take any PMC name as its type, but
that is wrong. It only accepts four types: `'Sub'`, `'Integer'`, `'String'`
and `'FixedIntegerArray'`. Don't ask me why these are the only four supported
from among all our built-in types, or why we support more than just 'Sub' (the
other three options seem superfluous to me).

The poorness of this syntax is not really something that I want to write about
in this post. We can do it better and in the next incarnation of PIR (if we
ever get to building such a beast) *will* do better. This is just one more
"of course it does it that way" kind of moment that you learn to ignore when
you're dealing with the code that is the current incarnation of IMCC.

What I do want to write about instead is the file where the parsing code for
this exists: `compilers/imcc/pbc.c`. That file contains much of the logic for
taking IMCC's internal `SymReg` representation and turning it into a packfile.
It's a huge mess, and encapsulation is broken here in ways that are as bad or
worse than any other single example in the repository. That's why I'm planning
a shotgun cleanup of it soon.

The functions and logical blocks in this file fall into two broad categories:
First, there are functions for iterating the AST (`struct SymReg` and friends)
and pulling out relevant values. Second, there are functions for inserting new
data into the budding packfile. The first category of functions is generally
fine and a necessary part of any assembler, even if some of the code could be
cleaned and modernized. It's that second category that's of much broader
interest. My plan is to take functions and sequences from `pbc.c` out of IMCC,
wrap them up all pretty, and add them to the proper packfile subsystem API.

Of course, I do start to think about how exactly to do that. At runtime
the `PackFile*` structure is basically read-only. Bytecode is read-only and
contains fixed integer indices into the constants table which is also not
expected to change. If bytecode isn't changing then annotations and debugging
info for that bytecode is probably not changing either. Once a packfile is
loaded into the interp, given a PackfileView PMC wrapper, and made executable
it really shouldn't be modified any more.

However when we're talking about a compiler or other code generating system,
we want the ability to write and modify packfiles. When we're done modifying
we might want to stamp them with a flag to say that they are read-only and
suitable for executation.

So I'm thinking we want two APIs. The first uses a bit flag on the packfile to
determine if it's editable, and can edit it if that flag is set. The second
is the normal read-only accessor API which can generally ignore that flag
except for the routines that load a packfile in to be executed by the
interpreter. For those handful of routines that do actual loading and
verification we can throw an exception or something.

My general plan, and I want to get a lot of feedback on this before I touch
anything, is to make a second API for packfile editing routines. I'll prefix
those functions with something like `Parrot_pfw_` (instead of `Parrot_pf_`)
to set them aside. I'll then start moving packfile building logic from IMCC
into this new API. Instantly we get much improved encapsulation, clear
separation of concerns between packfile writing and executing, and a more
robust interface for compiler writers to use in the future. It will also be a
nice development in terms of security, where we can limit certain packfiles to
be executable. I think it's a pretty good idea *and* I don't think it should
take too much effort to accomplish. At least, not in a first draft.

I'm not starting any new projects until I get my `remove_sub_flags` branch much
further along. I think it's a good idea to follow up a tough project with one
which is more straight-forward and offers clear rewards. Speaking of that
branch, parrot is building fine and I'm slowly working my way through the list
of test failures. When I get the majority of those sorted out I'm going to
start working on patches for Winxed, NQP-rx, NQP and Rakudo. We're a long way
off but the progress is extremely rewarding.
