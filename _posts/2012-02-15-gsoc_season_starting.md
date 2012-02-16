---
layout: post
categories: [Parrot, GSOC]
title: GSOC 2012 Is Starting
---

You might not see it, but I'm starting to get very excited. Discussions about
the Google Summer of Code program is starting up for the 2012 summer. Projects
in years past have lead to some awesome developments in Parrot, either directly
or indirectly, and 2012 could easily deliver more.

For prospective students, here are some blog posts I've written in years past
about the process, and what you need to do to get started:

* [GSOC Proposals](http://whiteknight.github.com/2011/04/11/gsoc_proposals.html)
* [Next Steps](http://whiteknight.github.com/2011/03/27/gsoc_students_next_steps.html)

If you're interested in participating in GSoC this year, with Parrot or any
other organization, I suggest you read those two posts. They're important. Many
students in school or freshly graduated may know how to code but might not know
much about some of the tangential topics: source control (git, in the case of
Parrot), documentation, unit testing, refactoring, etc. Now would be a great
time to start brushing up on all those topics, so you don't waste time during
the summer getting your tools in place.

As usual I will try to post some ideas for projects here on this blog. If you
are an eligible student and you are interested in one or more of these ideas
please get in touch with me or other interested Parrot developers. If you have
other ideas that I don't mention, that's cool too. Get in touch anyway and we
can start talking about those ideas. **The important thing is to talk to us**.
Seriously, it's important.

Here are a few ideas off the top of my head that might be worth some more
investigation. I might write additional posts about these ideas if people want
more information about them:

* **Opcode disassembler**. We have a growing set of tools for working with
bytecode from running Parrot code, but we don't have all the pieces that we
would need to make a fully self-hosted disassembler. What we still need are
tools to read out the raw bytecode and convert into a sequence of Opcode PMC
representations, then a driver program to turn that output into readable (and
hopefully round-trim compilable) disassembly output.
* **PACT** An alternate to Parrot's venerable PCT library, called PACT, has been
in the planning stages for many months. What we need is somebody with the time
and motivation to start putting those ideas into practice. A toolkit library
that works with syntax trees and outputs working bytecode libraries would be
quite an awesome thing to have, and would help us push the state-of-the-art
for Parrot compilers up a few notches. Intended to be a layered system, the
successful student could implement only a few of the necessary layers.
* **Jaesop Stage 1** Jaesop is a bootstrapped JavaScript compiler. Right now
there is a stage 0 compiler which uses node.js to compile JS into Winxed. We
need a stage 1 compiler, written in JavaScript, that can run on stage 0 and
compile itself. It's an interesting, mind-bendy kind of project. If you know
compilers and you like JavaScript, this might be the project for you.
* **Anything Python-Related** Parrot wants and needs more Python love. We've
had a few attempts at making a working Python compiler before on Parrot.
Working on any of those, starting a new attempt, or working on other ways to
integrate Python and Parrot would all be greeted with some eagerness.
* **New Object Model** Parrot needs a new object model. Right now we have 6model
waiting in the environs, but we haven't integrated it yet. There is some
question about whether we want to copy+paste merge 6model as it is, or if we
want to try and make a more custom adaptation of it from the ground up.
Proposals to do either, or anything closely related, would be very interesting.
* **LAPACK Bindings for PLA** I've wanted PLA to get LAPACK bindings for a long
time now, and I've never had the time to do it myself. I've never really even
had the time to design what such a thing would look like. I suspect we can do
the lion's share through raw NCI. Tools to solve matrices for eigenvalues and
eigenvectors and common transformations and decompositions would be very
interesting indeed.
* **New PackFile Loader** I've complained about the unnecessarily magic behavior
of our packfile loader before. A new, re-written packfile loader would not
automatically assemble Namespace or MultiSub PMCs at load time. Down this
path lay the possibility for significant performance improvements and complexity
reductions in some of our oldest and least-friendly code. It would require
serious changes to IMCC, PIR, and higher-level compilers like Winxed. The new
NQP already does the right kind of thing and would serve as a great example to
follow.
* **Anything Thread or Asynchrony Related** Parrot has Green Threads on Unixy
platforms. Extending proper support to Windows and elsewhere would be awesome
(again, something I want to do but have not had time or a Windows machine for
testing). Asynchronous IO using new threading primitives and anything else that
you could think to build with the new threading system would be awesome. Adding
in types that would help the effort such as lock-free arrays and hashes would
be nice assets. Adding in support for concurrency primitives like locks, mutexes
and critical sections would also be cool (and implementing those in terms of
green thread primitives would be even cooler).
* **Divided VTABLE** Parrot's VTABLE is a huge monolithic structure, and there
have been many suggestions recently that we break it down smaller chunks based
on roles. The "Number" role would contain arithmetic vtables, while the
"invokable" role would have VTABLE_invoke and friends. GCable role, Array role,
Hash role, Metaobjecet role, etc. These are all things that we could use and
would decrease overall memory footprint and increase flexibility in the system.
Bonus points if this work was integrated closely with 6model and other proposed
changes in our PMC subsystem.

These are just a few of the ideas I have on the top of my head this morning.
Some, I'm sure, are too big. Others are too small. But in each is the kernel of
a good idea and if anybody reading this is interested we should start the
conversation now to get these vague ideas focused into compelling proposals.

