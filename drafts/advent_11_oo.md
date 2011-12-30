---
layout: post
categories: [Parrot, Advent2011]
title: Advent 11 - Object Model
---

Parrot's object model is really not so great. That's putting things mildly,
but in the spirit of the holiday season I'll try to avoid saying all sorts of
insulting things like "Crap" and "Garbage" and "This is all very stupid". Lucky
for us, depending on how much work you can pile on your TODO list and still
consider yourself "lucky", we have a replacement in the environs: 6model.

6model is an object model designed to support the object semantic needs of the
Rakudo project. However, it's core design is simple and general enough that
languages far different from Perl6 should be able to use it to great effect.
6model allows us to implement various low-level object representations for
different purposes and helps to separate the form from the function of various
primitives to enable flexible use and reuse of components.

Parrot wants 6model. I've been saying that for months. We haven't really been
able to work on the port just yet, but we are definitely closing in on a pain
threshold where it's going to be the single most overwhelmingly important thing
that we need. So many problems will be solved, not just at the HLL level but
also internally to Parrot by the advent of 6model that it's almost futile to
try and name them all.

The question we have is exactly how we want to port it. I was planning on a
gradual approach where we more or less copy and paste the existing 6model code
into Parrot and provide it as a parallel offering while we integrate behind the
scenes. Some people suggested that we might do better following the 6model mold
and rewriting it from the ground up, but I'm not sure that effort buys us
anything that more gradual tweaking of existing code wouldn't also get us
eventually.
