---
layout: post
categories: [Parrot, Advent2011]
title: Advent 2 - Contributing to Parrot
---

I asked for some ideas about topics to write about for this advent calendar
thing, and PerlJam (Perl hacker and long-time Parrot ecosystem contributor and
well-wisher) suggested I devote a post to how to contribute to Parrot. Say no
more! I think it's an excellent idea for a second post. This, the second day of
my lazy, late pseudo-advent calendar for Parrot will be devoted to
contributions.

In the olden-days of Parrot, like last year, we used SVN. These were dark and
dangerous times when new contributors were forced to submit their proposed
changes in patch files via email. There was much weeping and gnashing of teeth.
However, we finally made the switch not only to version control with git but
also hosting with github. Now contributing to Parrot or any of its many
ecosystem projects is a snap. Create a fork of the repository you want to
contribute to, make your edits and commit them, and then open a github pull
request. It's so easy, you're going to be wishing you started doing it earlier.
Seriously, what are you waiting for?

If you submit enough cool changes, we'll probably do one or both of the
following:

1. Ask you to add your name to CREDITS, so you can get MAD PROPZ for everybody
   to see.
2. Give you a commit bit to the repo because seriously, you're doing too much
   awesome work and we don't want to have to play intermediaries between you
   and the code.

We may also try to rope you into doing other stuff, like being one of our
monthly release managers (looks *great* on a resume, especially if you're
entry-level), writing blog posts, mentoring GSoC and GCI students, and writing
even more awesome code. If you ever read the book [The Giving Tree][], it's kind
of like that except everybody wins and the ending is much happier.

[The Giving Tree]: http://www.amazon.com/Giving-Tree-Sid-Silverstein/dp/B004R64766/ref=sr_1_sc_2?ie=UTF8&qid=1323556649&sr=8-2-spell

Parrot is a relatively rare kind of project because we have so much happening at
so many levels of abstraction. If you're a nuts and bolts kind of coder and like
doing stuff "on the metal", we have plenty of internals work that needs doing:
Threading and Concurrency, Garbage Collection, Object Model, and more
optimizations than you can shake a stick at. If you're more of a middle-ware
person we have lots of libraries and infrastructure projects to work on. Then
we have HLL compilers like Rakudo that need help and finally end-user code and
programs written in those HLLs.

Want to write games? We have xlib and opengl bindings available. Sure they
may need a little bit of love, but we have them and they work very well.

Like compilers? We have a handful in development and are always looking for
more. Winxed is a fun and familiar (For C++ and Java lovers) systems language.
I'm working on a new bootstrapped JavaScript compiler. In the past we've had
Ruby, Python and Tcl compilers under development, and things

Like Perl6? RAKUDO RAKUDO RAKUDO! Also, Rakudo.

Like writing documentation or making cool new websites to hold our docs? We need
you! We've got lots of documentation that we need to expose to the users better,
and we are missing lots of documentation that needs to be written. Also, since
code is changing at a break-neck pace we need to update all the docs we have.

If you like solving problems, fixing bugs or implementing cool new code then
we have tons of jobs for you! Sign up for a free account on Github if you don't
have one already. Search around the various [Parrot repositories][] or search
for "Parrot" to find a repository you want to hack on. Create a fork and get
to work! If you want to contribute to a particular project or particular type of
project and aren't sure where or how is the best place, come talk to us and
we'll try to get you pointed in the right direction.

[Parrot repositories]: http://github.com/parrot

Speaking of talking to us, there are three good ways to get into contact with
us Parrot folks, if you want to chat: IRC (#parrot on irc.parrot.org), The
parrot-dev email list (parrot-dev at lists.parrot.org) or in comments and pull
requests on github. Leave a comment on this blog too, and I'll help you
personally.

Parrot is a big open-source project with lots of work to be done at every level
of abstraction. We're always looking for new contributions and new contributors.
If you're interested in getting involved, don't hesitate to get in contact
with us and start writing some great code.
