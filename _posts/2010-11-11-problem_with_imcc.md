---
layout: post
categories: [Parrot, Embedding, ParrotProductManagement]
title: The Problem with IMCC
---

Parrotteer Peter Lobsinger added a [comment][plobsingcomment] to my
[post from yesterday][embedapiproblems] that absolutely blew my mind. It
wasn't completely out of left field, but it was a difference in perspective
that really got me thinking about things in a different way. Here is his
comment (some editing for formatting reasons):

[plobsingcomment]: /2010/11/09/embedding_problems.html#comment-95966184
[embedapiproblems]: /2010/11/09/embedding_problems.html

> IMCC isn't a part of parrot.exe. It *is* parrot.exe.
>
> * perl test.pl # perl executable
> * perl6 test.p6 # perl 6 executable (rakudo)
> * parrot test.pir # pir executable (imcc)
>
> Yes, this is all kinds of broken. parrot.exe shouldn't really be IMCC. IMCC
> should be provided the same way as NQP, as parrot-imcc or parrot-pir.
>
> A few things fall out of this:
>
> 1. IMCC has no business being in libparrot. this includes attributes in
>    parrot_interp_t that are only useful to IMCC.
> 2. command line processing for parrot.exe should be managed exclusively by
>    IMCC. this is a good way to break things up as some command line
>    arguments only make sense to IMCC (and so need to be handled there).
>
> We may want to split the compiler of IMCC out into a separately loadable
> module, but that will be no small feat.

I had a vision of `parrot.exe`, the Parrot executable, being something that it
isn't yet. Peter is seeing it exactly how it currently is: IMCC as a frontend
to libparrot. The benefit to a proper API is that we can have multiple
frontends, and a PIR-compiling frontend (using IMCC) is the only one we have
right now. My primary goal is to make it possible to have others, we don't
necessarily need to worry about completely fixing the one we currently have.

In terms of my [current work][embedapiproject], this is a disasterous new
perspective. What I am doing is, in essence, two things. The first, and most
important is to properly encapsulate libparrot behind a nice, modern,
flexible interface. The second is to use that new interface to decouple the
Parrot executable from libparrot. IMCC likewise is bipartite: It has a
bison-based PIR parser, and a huge amount of code to couple that parser to
libparrot. Since nobody ever worked to design and implement an API on the
scale that I am doing now, and since nobody has drawn lines in the sand where
I am trying to draw them, IMCC has become extremely deeply coupled to the rest
of Parrot and there is no easy way to detangle the two parts.

[embedapiproject]: /2010/11/06/embedding_api.html

IMCC is a gigantic exercise in deep software coupling, and I am working on
attempting to decouple it from Parrot. In short: I'm screwed.

Well, I'm not entirely screwed. It is a doable project, and I am a determined
developer. Plus, several other people, including Peter and Parrot development
newcomer bluescreen (Mariano Wahlmann) have offered their considerable talents
to help. One more saving grace is that IMCC has a limited shelf-life.
Eventually, it will be replaced by [PIRATE][pirate]. If we procrastinate the
decoupling work for long enough, maybe we can outlast IMCC entirely, and
instead start fresh with a new frontend.

[pirate]: https://github.com/parrot/pir

Since I wrote my post yesterday, I've gotten a lot of feedback on some of the
ideas that I discussed. I appreciate everything I've heard from people, and
I want to encourage more people to take a look at the code and voice their
opinions. Some of the big decisions that I've made so far about the look and
operation of the API I'm pretty happy about and don't think I'm going to
change too much. But many of the smaller issues, including the form and
function of individual operations, are still up in the air and need lots of
help before I can say that they are firmly nailed down.

If you haven't seen the code yet, the majority of the new interface is in
[`src/embed/api.c`][embedapisource]. Usages of these new functions can be
found in [`src/main.c`][maincsource]. Take a gander at these two files and
read through some of the code. Obviously it's a work in progress, so things
will be changing pretty frequently. Check in often!

[embedapisource]: https://github.com/Whiteknight/parrot/blob/master/src/embed/api.c
[maincsource]: https://github.com/Whiteknight/parrot/blob/master/src/main.c
