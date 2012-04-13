---
layout: post
categories: [Parrot, GSOC2012]
title: GSOC Proposals Received
---

The GSOC 2012 proposal deadline has come and gone. We've received several
project proposals, although only half a dozen are serious, honest, plausible
proposals. We have a few days now to rank them, comment on them, and assign
potential mentors to them. When we find out how many slots Google is able to
assign to us, we'll be able to pick out which ones will be worked on this
summer.

Here's a list of the decent-looking proposals we've received:

* **Jaesop Stage 1 Compiler and Runtime** by **mayank**. mayank intends to
  fix up the last remaining bits of stage 0, then get started on a
  Javascript-compiler-in-Javascript stage 1. By the end of the summer I think
  it's very plausible that he could have a self-hosting compiler for most of the
  JavaScript language and at least a start on the basic runtime. If he keeps
  the abstraction boundaries nice and clean, after the summer is over we should
  be well primed to start upgrading bits of the internals to use 6model and
  PACT, when those two projects are ready to be used there.
* **LAPACK Bindings for Parrot-Linear-Algebra** by **jashwanth**. This is
  something I've wanted to add to PLA since the beginning of that project, and
  an absolute necessity if I ever want to get back to my dream of writing an
  M language compiler for Parrot. jashwanth has proposed assing LAPACK bindings
  to PLA (via NCI) and implementing a nice interface for some of the most
  important transformations, decompositions and operations provided by that
  library. He also intends to provide a few pure-parrot backup implementations
  for cases when LAPACK isn't available but we still need to get work done. It's
  an open-ended project that can be done in small, discrete chunks.
* **6model Integration** by **benabik**. We know we want 6model, and we know
  benabik has the chops to pull it off. He is still working on his thesis AND
  is expecting a baby this summer, but somehow I still don't feel like it's
  an undoable project. His proposal is to integrate 6model into Parrot's core
  and start transitioning our existing PMCs to use 6model instead (and abandon
  most of our current object model).
* **PACT Assembly** by **benabik**. Yes, benabik has submitted two proposals.
  This one is the start of PACT; something that, like 6model, we know we want.
  benabik, considering PACT was his brain child and he's getting his PhD in
  compiler-related topics, is uniquely qualified to pull this one off and
  make it shine. The real question is which one of his two proposals we as a
  community want him to work on more (I'm already personally signed up to
  do whichever one he doesn't pick, so it shouldn't be a loss either way). PACT,
  as I've talked about before, is intended to be a large modular library of
  compiler tools and building blocks, so there is ample room to expand the
  project if things are going unusually well.
* **Security Sandbox** by **Justin**. Security sandboxing is something that
  we've wanted, to varying degrees, for years. Justin has proposed to at least
  get us started with proper security and implement as many permissions and
  restrictions as he has time for. It's a project that we can consider to be
  a "success" if even half of what gets proposed actually gets completed, and
  there is plenty of room to build on if his momentum gets up.
* **Mod_Parrot 2.0** by **brrt**. ModParrot, the Parrot module for the Apache
  webserver hasn't been actively maintained in some time, and has fallen into
  disrepair following many of the internals changes to Parrot in the past few
  years. brrt has proposed an update to ModParrot to use the new and more
  stable embedding API. This is another modular project that can grow if his
  development speed stays high to include all sorts of helper libraries,
  driver programs, plugins for HLLs (Rakudo in particular) and other things.
  Most valuable at all may be his plans for implementing an automated test
  suite, which will help ensure ModParrot never falls by the wayside again.

So we've gotten 6 decent proposals from 5 students, and if even half of these go
on to succeed in reaching their goals Parrot will be much better off by the end
of the summer. And this list doesn't even include the calling-conventions work
that bacek is working on, or the threading work that nine is working on, The
M0 work that several other developers are doing, or the packfile and IMCC and
whatever else work that I'm planning for myself. This could be a very eventful
summer indeed.

If you're signed up to be a mentor this summer, or if you would like to be,
please head over to the [GSOC website](http://www.google-melange.com), sign up, and
take a look at the proposals.
