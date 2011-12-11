---
layout: post
categories: [Parrot, Advent2011]
title: Advent 1 - Parrot
---

I had intended to publish this advent post yesterday, but even though I had it
mostly pre-written I couldn't find 5 minutes in the day to do it. An
inauspicious start to this little sequence! It is already painfully clear that
this advent calendar of mine isn't going to be nearly as reliable or successful
as moritz's. To (finally) kick off this [pseudo-advent calendar][advent_main]
I'm going to jump right in and talk about the main course, the bird, Parrot.
This post is going to be a short retrospective about the developments in 2011
and some clues about where we are heading (or, where I hope we are heading) next
year. I'll very likely post a more in-depth yearly retrospective around the 4.0
release in January.

[advent_main]: /2011/12/08/advent_calendar.html

Parrot, as we all know, is a virtual machine aimed at running dynamic
languages. Originally it was envisioned as the backend VM for the new Perl6
language, but Parrot quickly deviated from that path. The idea quickly became
to create a language-agnostic platform for hosting a variety of languages
in a common, interoperable way.

I don't want to attempt a complete retelling of the history of the project. I
wasn't around for most of it and at best I'm going to give a faulty recount.
Regardless, it doesn't really matter how we got to where we are now. What
matters is what our current trajectory is. Prior to 2011 Parrot had been trying
to do many things well but did no single thing well enough to really drive
adoption. I think we've made up our minds to refocus our efforts on supporting
Perl6 specifically, and many of the biggest developments planned for 2012 are
going to be headed in that direction. Almost all of the work I personally am
planning to do next year will be directly tied to Rakudo, trying to make their
system even better, and to give them a better and more compelling infrastructure
to do their thing on.

2011 has seen a lot of changes to Parrot, though the vast number of them are
internal and involve code that is cleaner and more maintainable, even if not
much more functional or with better performance. This continues a trend that
was happening in 2010 and even earlier: The biggest tasks we've been working on
in the last few years involve trying to get some of our older, uglier, and more
brittle systems up to a decent level of quality to support additional future
work. It's the nature of the beast, and as much as I would love to say that the
code we had was perfectly pretty to begin with, in many cases it has not been.
This is not to disparage any prior contributors. If the history of Parrot tells
us anything, it's that Parrot hackers historically didn't (and honestly, might
still not completely) understand exactly what the goals were. So many decisions
intellgently made with the best of intentions lead in the exact wrong
directions. Such is life. So many systems were prototyped and those prototypes
silently became "the real thing" without anybody explicitly giving a stamp of
approval.

Starting several years ago we had many subsytems either deleted outright because
they were unsalvagable or dramatically rewritten. Some of those events were a
little painful, but in all cases we did what was necessary. Because of all the
hard work from our development team we are really starting to get to a point
where big system improvements and feature additions are not only possible but
now very plausible. Things that would have been near impossible to implement in
the code base as of 2009 are very reasonable to consider doing in 2011 and 2012.
That's a big step up, even if many of these changes aren't visible to the end
user.

Much of my personal work has been focused on getting some of the essentials into
place with regards to the core execution pathway: IMCC, Embedding API,
packfiles, etc. The new embedding API was added in January. IMCC's new public
interface was likewise improved and several bits of related code were cleaned in
April. The new PackfileView PMC type, the `load_bytecode_p_s` opcode, the new
bootstrapping frontend, and the new `:tag` syntax for PIR all came in the
second half of the year. Starting in early 2012 I'm going to be ripping out
all the old cruft that these things were designed to replace, and we are going
to see some improvements in code quality to go along with that.

Previously, users had the `:load`, `:init` and `:main` flags to try and schedule
when certain subroutines should be executed. The rules for all these flags were
messy and overlapping, and they still weren't considered enough for all
necessary use-cases. Some people had suggested adding even more subroutine flags
with more special-case semantics throughout the Parrot codebase. This, in my
mind, was nonsensical.

Now in Parrot you can use the new `:tag` syntax to tag a sub with any flag you
want:

    # Almost same as old :load
    .sub 'foo' :tag("load")

    # Almost the same as old :init
    .sub 'bar' :tag("init")

    # Couldn't do this before!
    .sub 'baz' :tag("SomethingNew")

And now with the new PackfileView PMC you can find and execute subroutines by
tag at any time you want without having to hope and pray that the magical Parrot
behavior will do what you need when you need it:

    $P0 = load_bytecode "foo.pbc"
    $I0 = $P0.'is_initialized'("SomethingNew")
    if $I0 goto already_initialized
    $P1 = $P0.'subs_by_tag'("SomethingNew")
    ...
    $P0.'mark_initialized'("SomethingNew")
  already_initialized:
    ...

Yes, it is a little bit more code for the end user (or intrepid HLL developer)
to write, but the increase in control and flexibility more than makes up for it,
in my mind. Plus, this kind of code is very easy to
[abstract away into a new function][init_bytecode], so you only need to write it
once. For the record, and I know I've discussed this topic at length on my blog
in the past, the above code snippet actually executes *faster* than the old
magical semantics, despite the fact that we have more explicit PIR code and more
code total running in the runloop instead of at the C level.

[init_bytecode]: https://github.com/Whiteknight/Rosella/blob/master/src/core/Rosella.winxed#L168

The [packfile loader][], PIR compilation symantics and the way things like
Classes, Namespaces and Multisubs are created will all be changing in 2012
for the better. Expect to see performance improvements and feature
additions to these things in 2012. If you're being smart and writing your code
in Winxed or NQP we will try our best to keep you shielded from almost all of
these changes. If you're writting code in PIR I feel bad for you, son. I've got
99 problems and PIR is like 97 of them.

[packfile_loader]: /2011/08/17/sub-first-steps.html

In a related note, a selection of other Packfile-related PMCs were refactored
in January and now we have the ability to create usable bytecode from a
Parrot-run program. The interface isn't great and we haven't made much use of
this ability yet, but I expect big things to happen in 2012 with [PACT][]
(benabik's rewrite of PCT) and maybe a few details coming to [Rosella][] as well.

[PACT]: http://github.com/parrot/PACT
[Rosella]: http://github.com/Whiteknight/Rosella

In 2012 I expect that we are going to finish the work we started in 2011 and
have IMCC removed as a permanent built-in part of Parrot. It will still be
available to people who want it as an optional library, but it will not be a
part of the libparrot binary itself. This is going to have some big
ramifications throughout some of the oldest and ugliest parts of the code base
and will allow us to start pursuing several goals: Decreasing total code size,
especially for embedded environments, being much more flexible about
compilers and cleaning up the compreg system, divorcing packfile creation from
packfile execution, giving us a much cleaner and more usable interface
for executing packfiles, and breaking up some of the syntactic quirks of PIR
from the inner semantics of Parrot. It really doesn't seem like much but trust
me when I say that removing IMCC represents more than just some theoretical
edification, it opens many doors that we *do* want and need to travel through
soon.

In March bacek added the new Generational garbage collector (GMS), and it
became the default collector in May. Performance jumped considerably,
especially for Rakudo; although I think some other optimizations can push
the performance numbers even better. Sometime in 2012 I want to test out an
idea I have with cutting out indirect function calls in the GC, and then I
want to dig through some profiles to see where else I can squeeze out a few
percentage points. I don't know if there will be much, but it never hurts to
look. With threading on the horizon, I also want to look into a few
concurrency-friendly GC algorithms that we can utilize to cut GC overhead and
maybe improve GC performance for heavily concurrent workloads.

Parrot hacker plobsing had been doing a lot of NCI-related work through the
year, adding the StructView, Ptr, PtrBuf and PtrObj PMC types to replace
some of the older types we had for working with raw pointers. These tools
are pretty low-level and don't always expose a very friendly interface (as we
would expect at such a low level of abstraction), but have a lot of
potential that we can definitely build on. Not all of the NCI changes were
met with great enthusiasm (especially those that broke some order signature
syntaxes), but getting NCI cleaned up and fixed up is a major hurdle for us
that we have to leap over in spite of the pain. Being able to share common
native library bindings among multiple HLLs is a huge deal for Parrot, and
has the potential to become a major selling point if we can get it done
correctly. The Rakudo/NQP folks have also been doing some cool-looking NCI work
that we may want to copy in whole or in part, so look forward to those kinds of
developments in 2012 as well. Hacker NotFound has been working on a new project
called [Guitor][] which uses pure [Winxed][] code to create graphical user
interfaces by calling into xlib using Parrot's NCI. The work he's been doing is
pretty fantastic, and serves as a clear demonstration that NCI is working and
working reasonably well.

[Guitor]: http://github.com/NotFound/Guitor
[Winxed]: http://github.com/NotFound/winxed

Adding new native library bindings for Parrot is a very productive and very
informative way to get started hacking with Parrot, for anybody who is
interested!

Parrot's object model has remained relatively static for a while, despite the
fact that the problems and limitations with it are well known and oft-decried. I
hard started to port 6model, the new Rakudo object model developed by Jonathan
Worthington, over to Parrot in the summer months but put that work on hold while
I came up with a better plan. I kept hoping that development on 6model would
slow down and become more stable so I could find a good jumping off point to
start with, but that never happened. Eventually I am going to need to just
put my foot down and start moving code around. Expect 6model to come to Parrot
in early 2012, especially if a few other people volunteer to help.

[Winxed][], NotFound's system-level language for Parrot was added as a snapshot
to the Parrot repository in July, and has continued to improve by leaps and
bounds in that time. Recently NotFound added syntax for `inline` functions,
which can help to improve performance in many places. I've got a few syntax
ideas I want to play with and try to contribute as well. If we can divorce
Winxed a little further from IMCC and PIR syntax (especially in the area of
calling conventions and flags), we can be more free to change Parrot's
internals because Winxed's abstracted syntax will create more of a buffer for
us. In 2012 I would like to continue the trend of using PIR less and less, and
using Winxed and NQP more and more for basic Parrot tasks.

Specaking of which, NQP-rx, an older variant of the amazingly useful NQP
language is showing it's age and may be dead in 2012. I would love to see it
replaced by the newer 6model-powered [NQP][], especially once we get 6model in
Parrot natively. Then we will have two awesome lower-level languages to play
with in an interchangable, inter-usable way.

[NQP]: http://github.com/perl6/NQP

Late 2011 also brought us nine's rewrite of Green Threads. They aren't working
on Windows yet, but they are working reasonably well on Linux and (I think)
Mac although some kinks are still being worked out. In early 2012 expect to
start seeing his implementation of full hybrid threads which are already
looking awesome. As with the green threads Windows support may come later,
especially since we seem to have a relative dearth of hackers working on that
platform right now. When I finally get my new laptop I'll keep windows around to
dual-boot from, and will do my best to get concurrency working as well there as
anywhere. As always, help in this endeavor would be appreciated. If you're
interested in Parrot concurrency, be it implementation of the internals or
using it to write cool new programs and libraries, definitely let me know.

Also in 2012 expect to start seeing some of my proposed changes to PCC start
getting integrated into the system. Actually, you probably won't see them,
most of the changes will be transparent in IMCC or maybe in new optimized code
generated by Winxed and NQP. We'll definitely see a few percentage points in
performance improvement across PCC calls to start, and opportunity for further
improvements after that.

This little retrospective has already grown into a substantial post so I won't
write too much more. Expect other posts in this advent series to be shorter
and sweeter (and hopefully, more on time).
