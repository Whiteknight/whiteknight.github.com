---
layout: post
title: Trials of the Product Manager
categories: [Parrot, ParrotProductManagement]
---

Yesterday in the Parrot chatroom we had some very difficult and occasionally
distressing discussions. The problem, if I may radically over-simplify, is
that several important contributors have been burned out in the past, and
several are in the middle of becoming burned out right now. The most common
complaint I have heard today and in days past is that Parrot development
happens too quickly, and the maintenance burden of compiler and extension
projects to keep up with the changes is overwhelming to smaller teams.

I never thought I would have to think about Parrot development moving too
quickly as being a problem. The change in perspective between core developers
and compiler/extension developers is a huge one.

An important root cause of the problem is that Parrot hasn't ever really had
an "official" API. To compensate for the lack of proper functions, many
projects are forced to bypass encapsulation and dig into the deep internals
of libparrot to perform even basic tasks. Even with our stodgy deprecation
policy, changes to our internals are fast and furious.

Quick recap: Some parts of Parrot change too quickly, and large changes to
the public "API" (so far as we have ever had such a thing) are made according
to the deprecation policy. These breakages are overwhelming to small projects
and one-man "teams". These developers spend much of their time fixing
breakages, and aren't able to keep up with other development tasks. This,
combined with the slow pace of fixing certain bugs requested by external
project developers in libparrot itself leads to a bad case of burnout.

Here's a scenario that's happened several times since I've been a member of
the Parrot project:

A developer is working on a pet project, and runs into a problem. Either
something has changed in Parrot or they have uncovered an as yet undiscovered
bug. They create a Trac ticket to discuss the issue. Maybe they get a little
feedback, maybe not. All the while the issue is not being fixed in Parrot,
development cannot continue in the project. Eventually, through a mix of
frustration, hopelessness and/or anger, the developer gives up entirely.

When I think back on all the cases of people who have gone through this hassle
and are no longer part of the project or who have taken a radically
diminished role I get pretty upset. It's not anything that should have ever
happened in the first place. If I can help it, it shouldn't ever happen again.

I want to start taking steps to correct this situation, though it is going to
take a lot of work and a lot of effort on my part. It needs to be done though,
and if I can't take steps to fix these things I'm hardly worthy of the title
of "Product Management Team Lead". I won't guarantee any miracles, only that
I will give it my best shot.

To get started I've taken ownership of all outstanding, unowned tickets
from HLL projects. I have already put together prototype fixes for
[two][firstticket] [tickets][secondticket], and as soon as I can get some
tests written for the behavior I can put the finishing touches on them. I
won't be able to resolve all these tickets single-handedly, at least not in a
timely manner, so I am hoping to be able to identify other developers who
would be willing to adopt some of them.

[firstticket]: http://trac.parrot.org/parrot/ticket/731
[secondticket]: http://trac.parrot.org/parrot/ticket/1028

In [trac][] I have set myself as the default recipient of any ticket marked
"docs", "extend/embed", "hll_interop", or "language". Now, if you create a
ticket with one of those components specified, it will be automatically
assigned to me. This should be an improvement over the situation where a new
ticket wouldn't be assinged to anybody, and thus garner no developer
attention.

[trac]: https://trac.parrot.org

I am also kicking around an idea of having regular or semi-regular meetings,
similar to #parrotsketch or a Parrot Developers Summit (PDS) specifically for
Parrot's users. The goal of such a meeting would be to create a standard
forum for gathering feedback so that I can identify pain points and take that
information back to the Parrot developer community for resolution. The Parrot
developers are having a PDS meeting on December 5th at 2300 UTC, and I am
heavily involved in planning for that. I won't be able to do any work on a
Parrot Users Meeting until after that, but if I get some good feedback I will
try to set something up shortly thereafter.

I'm doing the things that I think a successful Product Manager needs to be
doing. If anybody can think of other things I could try to further my aims, I
will be very open to ideas. If anybody likes the things that I am doing and
wants to join in with the work, let me know and I can add your name to the
team roster.

