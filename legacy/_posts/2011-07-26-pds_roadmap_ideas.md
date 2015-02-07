---
layout: post
categories: [Parrot, Roadmap]
title: Parrot PDS and 3.9 Roadmap
---

It's official, we're buying a house. Last night I got the last of the down
payment together in the form of a certified check. It's extremely difficult
seeing several thousand dollars of hard-fought money being taken out of the
savings account in a single fell swoop. Difficult, but at the same time
extremely rewarding and reassuring.

Settlement is happening on Thursday afternoon. From now until then my free
time is allocated for packing, planning and cooking. We're cooking meals that
can be easily packaged, frozen, and reheated later. These are not only for us
on days when our kitchen sundries are all trapped in boxes, for days when we
may have to make some necessary kitchen improvements (We bought a house
without a dishwasher!), and also for my in-laws. My wife's mother is having
surgery early next month, and we want to have some pre-made food available to
help make the recovery period easier. Anything I can cook and freeze today
will be a huge help in the days ahead.

Of course, the victim here is the free time I would normally spend on other
stuff: Coding, blogging, chatting on #parrot and searching for funny pictures
of cats on the internet. All of these things will take something of a back
seat for the next few days, and my availability in general will be down. This
includes for the Parrot Developer Summit meeting we're holding online this
coming weekend. We haven't scheduled a definitive time for the meeting yet,
but I have to make the conservative assumption that I won't have 2-3 hours
at any point during the weekend to sit focused in front of a computer. Maybe
we move so much stuff during the mornings that we are exhausted and need to
vegetate in the evenings. Maybe, more likely, the hard work will start as soon
as the boxes are all moved in to the apartment but not yet unpacked. I'll do
my best to attend the meeting at least in part, but I make no promises.

Jim Keenan sent out an email to the parrot-dev list asking for people to start
thinking and discussing the kinds of things we want to talk about at the
summit. In a general sense, the summit is the time when we lay out our
development roadmap for the next few months. The development roadmap is the
list of things that we are committed to deliver, and have assigned people to
work on. It is not, emphatically so, a list of things we would like to have,
or a list of things that would be really cool to have. The roadmap is only
for the things that we *need*, are *capable of delivering* in that timespan,
and for which we are *committed to deliver*. By necessity, the roadmap has
tended to be a small list of things, but our track record in delivering them
has been pretty good since the new system started. Much better than the old
system where the roadmap was little more than hopes and dust, thrown into the
wind.

So, since I might not be at PDS in person, I want to lay out my ideas
for these questions in blog form. I'll talk about the things I think Parrot
can and should deliver in the next three months, and which projects I would
be willing to be assigned to.

### GSoC

GSoC is going very well, and several of our students are on schedule and are
generally kicking some code butt. They do need our continued support, however,
to ensure that they continue onward and upward and that we get good results
at the end of the summer. This isn't the kind of specific goal that will end
up on our roadmap, and it's not the kind of thing we can really assign to
a person. What I want to see is that the students continue to get excellent
support from the developer community and that they are given the best
opportunities to succeed by the end of the summer.

### 6model

If you look at the benchmarks comparing the Rakudo nom and master branches,
what you see might be a little surprising. The "nom" branch, which is the
migration of Rakudo to 6model (among other changes) is significantly faster
than master. There are a few reasons for this, the most compelling of which
is that 6model allows native-typed attributes. Parrot's current Object
implementation boxes all attributes into PMCs, which is extremely costly and
wasteful. 6model also brings in some other efficiencies too, and enables
certain optimizations that we just don't have available in Parrot today.

We want 6model. I want it. And, most importantly, I'm extremely eager to do
the necessary work to bring it into Parrot. Given what we know today about the
problems with our current object model and the extreme improvements 6model
offers, everybody else should be chomping at the bit to make it happen as
well. Getting 6model moved into Parrot, and doing it as quickly as possible,
should be a big priority, and I would love to see it added to the roadmap.
I would love to spearhead the charge, and I think there are a few other
developers who would love to help it as well.

In fact, I'm probably going to assemble a team and work on this project
whether it's on the roadmap or not. Plan accordingly.

### Profiling

Thinking about profiling was on our roadmap leading up to the 3.6 release, and
we did the bare minimum of that. Christoph, who probably knows the most about
it, was busy with Lorito and I, who knows nearly nothing, wasn't really able
to do much in the way of productive advancement. Strictly-speaking, the road
map goal was to think about profiling, figure out what was wrong with it and
come up with a way forward. Actually implementing some of those things should
probably be on the roadmap for this quarter.

6model is going to bring some performance improvements throughout, at least
for languages and libraries which aren't currently using 6model. Rakudo for
instance probably won't see much of a bump, except in streamlining a few
corner cases. What will help to benefit Rakudo is adding in some improved
profiling tools, so they can find and eliminate bottlenecks. I think we really
need to work hard on this issue. I can't say I'm as eager to work on profiling
as I am to work on 6model, but I am *willing* to do it if hands are needed and
if suitable direction is provided. I won't be designing the new tools, but I
am willing to help implement them, since they are so important.

### General Optimization

There's a branch floating around to improve tunings for the GC. PCC is ripe
for optimizations and improvements. Several pending deprecations could bring
improved code and better performance. PMC- and Object-related code could be
significantly streamlined and improved, both before the migration to 6model
and afterwards. IMCC could stand to see some optimizations and removal of more
cruft. Besides these big areas, there is potential for a lot of streamlining
in various hotspots, spread throughout the codebase. I think it's reasonable,
and in fact it's becoming necessary, that we start to focus on optimizations.
I would like to see Parrot 3.9 be *at least 10% faster* on an assortment of
benchmarks on a variety of machines, than 3.6. With a concerted effort and
dedicated developer resources, I think we can make it happen. We do need to
pick the platforms that we want to target, and the benchmarks we want to
track, but that could be done in a week or less. We could write up several
benchmarks in NQP and Winxed, both of which would be significantly easier than
writing up benchmarks in PIR. Plus, we would be able to get a more
comprehensive measurement, including the speed of Parrot execution and the
quality of generated code in those widely-used tools.

I am definitely willing to spend some significant amount of time on
general-purpose optimization, and the more developers we can devote to this,
the better. I would love to see something like this end up on our roadmap,
and I think our users would love to see it as well.

### Infrastructure

This is an ancillary request. I would really like to see some serious thought
put in to various bits of the Parrot community infrastructure.

Our Trac installation doesn't have anything like a captcha or other limiting
tools to prevent spam, so we've been preventing arbitrary users from creating
tickets for bug reports and feature requests. That's a horrible situation and
needs to change. Since it appears to me that there aren't many good
alternatives, that Trac development appears a little anemic, and since
upgrading these kinds of things to get new features tends to be a royal pain,
I would be perfectly happy to move off Trac if an alternate idea were
presented. I'm not pushing this agenda, simply stating that I wouldn't shed
any tears if we left trac behind. Also, Git integration with Trac is
apparently broken now. I don't know how much of a hassle that will be to fix.

Other things could use a look too, even if only to verify that they are what
we need: The buildbot infrastructure, smolder, the mailing-lists, the various
chat bots that help with our IRC chatroom, the parrot.org website, and the
various steps in the monthly release procedure and the things that are
required by it. It's good for us to take inventory, and make sure that we are
getting the most utility for the developer hours that we spend to set up and
maintain these things.

This isn't a road map goal per se, but it is something that I think the Parrot
community should take the time to consider in the coming months.

So those are five things that I think we need to focus on between now and 3.9:
GSoC, 6model, Profiling, Optimization and infrastructure. Those are just my
ideas, and I know other developers are going to have other things as well. Of
roadmap items that I am interested in working on, 6model definitely tops the
list followed by optimization and then profiling.

If I cannot attend PDS, please consider this list of suggestions of things
that we should put on the roadmap, and this is me volunteering to work on any
and all of them.
