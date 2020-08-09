---
layout: post
categories: [Architecture, Business]
title: Being an Architect
---

I spoke to a recruiter about an Architect position recently. The position was described as "40% hands-on development". So, I asked him, "what is the other 60%?" and he responded, "well you tell me, you're an architect now, aren't you?"

Sure I am, but my job is almost 95% hands-on coding partly because it's what my organization needs but largely because of my own personal philosophy on the job. There are several things that are associated with architecture by some organizations, which I believe get over-valued or misunderstood. These include documentation, diagrams, research, proofs-of-concept and code reviews.

## Documentation

It's my experience that **Documentation is the first lie**. From the developer point of view, you already have a complete and thorough description of what the software does: the source code itself. Properly written, the source code should give all the necessary instruction to the computer while also being able to be read, understood, and maintained by other developers. ("properly written" is subjective of course. A piece of software is understandable only if people who didn't write it can understand it. For verification, ask a peer to read your code to see if they can follow along. Take whatever answer they give you as immutable fact, even if you disagree. After all, they are the reader, and you are measuring readability.)

The result of adding in documentation, especially extensive and "complete" documentation, is that you end up with two descriptions of the software. Separate is never equal, and two descriptions of the behavior will end up diverging over time in subtle ways. Maybe the two aren't updated together, or maybe the updater doesn't have the same skill with written English that they have with writing code. In either case there are going to be differences and when documentation differs from the source code, the source code is always right.

At this point it's worth differentiating between two types of documentation: the internal and the external. The external documentation is for users and downstream developers, and needs to cover the relevant bits of the external interface so that integrations can be written and work can be done. Non-technical readers won't read much of this but downstream developers *might*. I've found that, given perfectly good technical documentation, many downstream developers will instead opt for Google searches and StackOverflow questions anyway.

The internal documentation, documentation the development team writes for itself, is often in even worse shape. Without external readers there's no serious impetus for quality control. There's no deadline at which the documentation must be complete and correct or the organization as a whole ceases to function. For this reason people aren't motivated to write it or update it. When people realize how out-of-date the documentation has become they stop reading it, which leads to even less motivation to write it.

Documentation is almost always a lie. It lies because it becomes incorrect over time and it is a lie that people expect the team to read documentation when they usually don't.

## Diagrams

I've done my fair share of UML diagrams and dependency diagrams, code flow, database schema diagrams, and all sorts of other pictures, both pretty and informative. No matter how good they looked or how much information they contained, they were almost always completely worthless to the organization. Diagrams, after documentation, are the second lie.

Unlike documentation, which often only requires a text editor and basic knowledge of a keyboard and the English language, diagrams often require specialized software. Maybe it's Visio, or some UML editor or even a vector graphics package. In either case, some people will have it and other people won't, so the number of people who can participate in the diagramming effort is smaller than those capable of writing documentation.

An architect creates a diagram for what a system should look like and hands it off to the developers. "Do this" she says and, laying her finger aside of her nose, and giving a nod, up her ivory tower she goes. The developers examine the diagram closely and say "well, I guess we will do our best" and end up with a system which is almost completely, but not entirely, different. Oh, and by the time we've identified the discrepancy there's no time left for a rewrite. Now we have a diagram which was never really followed and a system which is completely undiagramed. This is not a value add.

The counter-examples I can think of are rough diagrams drawn at the beginning of a project to help with initial layout and resource allocation, and diagrams which are drawn at intervals so the overall architecture of a project can be easily visualized so course-corrections can be plotted. Both these types of diagrams should be at least hidden in an archive if not deleted outright shortly after use because of how quickly they will fail to represent reality.

There's also a point to be made about how many inexperienced developers who could most benefit from a succinct visualization of a complex system probably don't know UML or other diagram symbology well enough to use them effectively.

## Research and Proofs of Concept

The team has a new challenging problem to solve and it's natural to hand this off to the Architect to do some research. So the Architect compares libraries and systems, draws a few block diagrams, and creates a Proof Of Concept (POC) which the developers can follow when they do the actual work.

This can be extremely useful because often-times getting started with a new piece of technology can be tricky. You want your most experienced minds doing this kind of work: learning the hard lessons, identifying helpful patterns, and identifying the common pitfalls so your development teams can become more productive more quickly.

There are two problems that I see with this workflow, which many organizations fall into:

1. The POC is considered "good enough" and shuffled into production directly despite the quick-and-dirty nature of it's development. This often happens when an ambitious but non-technical manager sees an impressive tech demo and wants it right now, or when a self-absorbed Architect thinks that they are above the errors of more "common" minds and finds a way to skip the normal quality control pipeline.
1. The POC is handed off to the development team without further guidance on patterns or best practices. Developers use this as basically a form of documentation (and documentation is a lie, see above) and don't learn any lessons of value from it.

It should be the job of the Architect to not just show how something *does work* but instead how it *should be used*. This, like almost everything else in software development, is an iterative learning process. The best answers are almost never found on day one, but are instead approached through many cycles of trial and error.

## Code Review

I've said it before and I'll say it again: So many organizations get things backwards. Experienced senior devs, team leads and architects spend more and more time away from the keyboard where they are most productive and valuable. Instead, they spend time reading and reviewing code from the younger and less experienced developers. The people who bring the most value to the codebase aren't coding and the people who introduce the most bugs are coding almost 100% of the time! This is absurd practice and organizations which do this (again, almost all) get the results they deserve.

Imagine a classroom where the teacher only grades tests but never gives lessons, and the students are in charge of trying to teach themselves according to their own home-made lesson plans. It's absurd to think about and the results would be awful.

Instead, I firmly believe it should be the opposite: Your experienced developers should write as much code as they can, which your junior team members should read, follow, and attempt to replicate.

I had a ticket one time which required adding a new method to an interface with a large number of implementations. The work wasn't difficult so much as tedious, but if done right it could help solve a lot of problems throughout the codebase. So I personally did the first few additions and committed my changes. When I handed the work off to the other developer I gave her a few instructions:

1. I gave her a link to my commit along with instruction about what I did and why
1. I asked her to make a small number of similar changes and submit them for review.
1. When I reviewed the first few and saw that she was following the correct pattern, I had her complete the rest of the task.

In this way I wasn't having to review a huge Pull Request, see that everything was done wrong and send it back for revision. I was able to show her the right way to do it, make sure she understood the pattern, and then send her on her way with much more confidence that it would be completed correctly.

Contrary to popular belief, junior-level developers don't learn much from spending hours doing things the wrong way and then being told it's wrong at the end. I don't know why so many organizations operate this way.

## It's the Code, Stupid

An architect shouldn't be tossing diagrams, documentation or POCs over the wall like some kind of explosive weapon and hoping for the best. I think an Architect should be writing the actual code, and creating the actual system, and handing a working prototype or skeleton to the developers who instead can see exactly how the system should work and can more easily follow along. At the very least, the Architect should be a hands-on resource, either permanently embedded or semi-permanently attached to a team.

I've done open-source work, I've been a code grunt, a team lead, an architect, and everything in between. The most important lesson I've learned about people and processes is this: **Code speaks**. Opening up a Jira ticket to "please add this feature" is worth nothing and is easy to ignore. Submitting a Pull Request or patch with working code is worth a heck of a lot and is very hard to say "no" to. The same goes for everything in development: Good, high-quality code is what we want, and it's the responsibility of the Architects and other senior staff to produce it.

## Summary

So to summarize, an Architect should:

1. Produce code, especially the central, critical (performance-critical, security-critical, stability-critical, etc) bits of code that your system relies on
1. Use the code they produce as a model to help inform and guide other developers on the team
1. Do research on new technologies, libraries and systems, and
1. Find and disseminate patterns and best practices, via working code examples, to the rest of the team
1. Teach junior members with working code examples NOT post-facto code reviews, how to do things correctly

Maybe there are other ways to do it, but this is a large part of what being an architect means to me.
