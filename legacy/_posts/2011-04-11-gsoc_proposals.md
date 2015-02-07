---
layout: post
categories: [Parrot, GSOC]
title: GSoC Proposals
---

The proposal deadline for GSoC 2011 has come and gone. All the proposals that
we are going to receive have been received, and anybody who missed out on
the deadline are just going to have to wait until next year. Parrot has
received a grand total of 15 proposals from 14 individual students, some of
which are extremely impressive.

However, by this stage in the program I find that I'm already exhausted from
it. I've spent the last few days reading proposals and giving feedback, in
many cases the same exact feedback, to all of the students. There are some
issues and mistakes that every single student has made. Other issues that
were extremely common. There are a few issues that seem to be common to the
batch of last-minute proposals we received in the hours and minutes before
the deadline. In this post I want to talk about these various problems and
issues, so that we have something written down for next year. Obviously,
nothing I write today is going to help this years students, but it's only
after going through the application process as the organization admin for
Parrot that I've been able to distill some of these conclusions and opinions.

## The Code Is Only a Small Part

The single biggest, most common criticism of the proposals I've
received--every single one of them--was an almost complete omission of
documentation and testing. I could go off on a tangent and start complaining
that schools aren't teaching these valuable aspects as well as they should be,
but that sort of misses the mark. Many of our students aren't in CS programs,
so we wouldn't expect best practices related to coding to be taught to them
anyway.

Regardless, that's academia and this is the real world. Out here, in the
bright light we do a lot more than writing and rewriting merge sort
implementations. GSoC is all about taking your coding skills up to the next
level where academia doesn't go, and a big part of that is by understanding
the whole software process as it's actually practiced.

Working software is one thing. Properly documented and tested software is
something else entirely. Documentation and tests are as important as the
working software. Tests prove that the software works as expected and that it
doesn't break when things change in the dependency chain. Documention shows
how to use the software, and maybe even how to modify it. For a student whose
participation in Parrot is likely to be fleeting, these things are extremely
important. Documentation and testing are the only ways that we can guarantee
that your work will be and will remain useful to the community long after
you have moved on to other projects and other responsibilities.

When the summer ends and the student disappears forever, we need to know that
the code will be usable and maintainable by other people. The way to do this
is to provide tests which act as examples of proper execution, use tests to
indicate when something is broken, and provide documentation to help get new
coders acclimated to the project quickly. Without these things, the software
really can't be considered reliably useful to Parrot, and we are much less
likely to invest our time and energy mentoring you to produce it.

For your GSoC proposal, you should be explicit about your documentation and
tests. Mention what documentation tools you're going to use. Mention what
test tools you're going to use. For goodness sakes, don't try to homebrew
either of them (unless that is the project you are proposing). Use existing
toolchains where possible, and try to pick ones which are standard for your
problem space. For instance, if you're running on .NET, you could pick NUnit
for your tests, or one of a number of other acceptable options. It would not
make a hell of a lot of sense to use Test::More from the Perl5 library to do
it. If you're writing Python code, maybe you want to use doxygen for your
documentation. You probably don't want to use javadoc.

Take the time to research this. If you don't even know which tools are and
are not appropriate to use for things as fundamental as these, it suggests you
may not have enough background knowledge to complete your task adequately.

## Get Involved

I say this all the time, yet so many students seem to take it as a suggestion
which can easily be dismissed. When we are evaluating proposals, we do so with
an eye towards completability and chances of success. Can you, the student,
actually complete the work you propose in the time allotted? There's no way we
can predict the future, but we can look for some indicators to help us make
better predictions.

One point to keep in mind is that GSoC is not a charity program where FOSS
projects donate their time and their effort to selflessly mentor students.
Google is the generous benefactor here. Parrot is looking to *get something*
in return for its involvement. The student gets valuable real-world coding
experience (and money!), while the Parrot Foundation and other FOSS
organizations receive the gift of code. Your proposal should reflect this
reality. Be sure to tell us what we get from you, how we benefit, and why we
should invest in it. It doesn't matter to me if you think your proposal is
"easy" and that you are capable of completing it. It matters to me if you can
actually deliver (which is not typically a question left to self-assessment),
and if we would even want what you are offering.

We've received more than a handful of proposals for projects which did look
interesting and did have obvious benefits for the student, but provided no
benefit to Parrot. Some of the project ideas did not even require Parrot or
it's library ecosystem at all! It's extremely hard for us to get excited about
a student using software we didn't write, writing code in a programming
language which doesn't run on Parrot (yet), and performs a task that we don't
need to have done. Please keep this in mind before submitting proposals. You
should spend some time making certain that your idea is mutually beneficial.
You should be able to explain *how* it benefits Parrot.

We ask students to discuss their backgrounds on their proposals, but there is
only so much you can learn from this. Most students don't have huge online
portfolios so early in their careers for us to look at. Some of them have
Github accounts we can look at, but most don't. I suggest you do start putting
together portfolios of this sort if you are serious about a long-term career
in programming, if you haven't already, but that's not really a matter for the
topic at hand.

The single best way that we can get an understanding of your capabilities as
a student is to interact with you personally. You *must* come chat with us.
I don't care if it's over personal email, or on a mailing list, or on IRC.
One way or another you *must* get in touch with us. This isn't optional.
Re-read that: This is not optional. Absolutely. Not. Optional. I cannot stress
this point enough. We  **will not** accept an application from a student we
haven't spoken to, regardless of the quality of the proposal.

What are your plans and designs? Are you open to suggestion and direction? Are
you easy and even enjoyable to work with? Are you really familiar with the
topics you claim? Do you honestly understand the problems you are trying to
solve? Are you prepared for some of the potential issues that you will run
into during your work? Do you have backup plans? These are all the kinds of
things that we need to talk to you about in advance.

And on the flip side of that coin, you need to talk to us to get information
about us. What do we need? What do we want? What do we get excited about?
If you are trying to propose a project to benefit Parrot, this is valuable and
necessary information for you to have. If you propose something we don't want,
we won't accept the proposal. The only way to find out what we really want is
to ask us about it. You aren't going to find enough depth of information in a
short blurb from an online list of ideas. Ideas are great as a starting point,
but they absolutely are not enough.

But here's the funny thing. Talking to us is a great way to get feedback on
a draft of your proposal. The students who have been working with us the
longest, some of whom appeared in our chatroom the day it was announced Parrot
would be participating in GSoC, have the best proposals. This is because they
have had all the extra time to chat with us, find out what kinds of projects
we want, and get feedback from us about how to make those proposals better.

Some students came to us with lacklustre proposal ideas, but after several
iterations of feedback, and after discussions with Parrot contributors about
what *parrot wanted*, those proposal ideas were improved. Again, I mention
that several of these students now have some of the most impressive proposals
I have ever seen.

On a side note, the same thing goes for GSoC mentors: If we don't know who you
are, you aren't going to be a mentor for us. There's no negotiation on this
subject. Being a mentor involves non-trivial responsibilities for both the
student and the Parrot Foundation. This isn't the kind of thing we entrust
to just anybody, and it would be a disservice to everybody involved if we did
otherwise.

## Timeline

The timeline is the single most important part of the proposal. The timeline
serves several purposes:

1. It shows that you have a real plan. Saying "I'm going to write code until
   it works, and hopefully that happens before the deadline" is not a real
   plan.
2. It shows that you understand the work involved, and are able to estimate
   it.
3. It shows that you are familiar enough with the task to break it down into
   reasonable chunks
4. It shows that you are working on short timelines and small milestones,
   which are easiler to accomplish than a single large deliverable at the end
   of the summer.
5. It gives us an easy way to keep track of your progress once the summer has
   started, and evaluate how well the program is going for you.

Other organizations might be different, but here is what we want in a
timeline:

1. Break it up by weeks at least (smaller time periods are fine too, if you
   want to provide that level of detail).
2. Each week should have specific, explicit milestones. You should be able to
   say what you are going to be delivering and able to demonstrate at the end
   of each week.
3. You should include documentation, test, and build infrastructure milestones
   in your timeline. **Do not save these things until the end**.
4. You should include a prioritized list of additional items that you will
   work on if you are ahead of schedule.
5. You should include a prioritized list of items which you can reasonably
   cut if you are running behind schedule (This **DOES NOT** include
   documentation and testing).

Timeline entries like "Week 3: Keep working on Foo" are not specific enough.
If you have continuations like that in your timeline, it means you need to
break your big tasks down into smaller tasks. If you have an entry like
"Week 5: Fix bugs", that means you are planning to have bugs, or you are not
planning to be fixing them as you go, or you aren't doing testing early
enough to find the bugs earlier. Create mechanisms to help find, diagnose,
and fix bugs earler, and don't devote large blocks of time to it in your
proposal. This is why testing is so important. Testing helps you find bugs
before every commit, long before they pile up and require an entire week to
fix.

## Documentation And Testing

Seriously. I can't reiterate this enough. Learn it. Live it. Love it.

Documentation and testing are extremely important. They should become integral
parts of your daily coding sequence. Some people do test-driven development,
where you write a test first, then write the code to satisfy the test. Write
a test, write the code. Write a test, write the code. Continue.

Some people do it the other way. Write code, then write a test to prove it
works. Write the code, write a test. Repeat.

Get your bugs involved too. If you find and fix a bug, write a test to prove
that it is fixed and that it never comes back. This can be part of the fixing
process, a passing test is proof that the bug is fixed.

Some people get documentation into the mix as well. Each week, write draft
documentation for the features you are going to work on that week. This makes
sure you have an idea in mind and have planned how it is going to work. Then,
in short iterations, write your code and tests throughout the week. At the
end of the week, review the documentation to make sure it's still accurate,
changing whatever needs to be changed to make it accurate.

Mention on your proposal that a particular feature is not "complete" until it
is properly tested and documented. Working code that is not accompanied by
tests and docs is not complete. If you get to the end of the week and don't
have these things, you have missed your milestone, even if "the code works".

Documentation and testing are not just add-ons, they should be an integral
component of your development process. These things are tools, and if you
use them properly, they can help make your work faster. Even if you don't
learn to leverage them for their productivity benefits, they are still
required deliverables. If you're doing the extra work anyway, you should
probably try to figure out what benefits can be derived from them.

If you don't understand how to properly use testing in your application, or
how to properly leverage documentation, ask. That's what mentors are for.

## Write, Edit, Revise

I learned this sequence back when I could still count up to my age on my
fingers. Proposals are important, because if we don't get as many slots as
we get proposals, we have to start making decisions. Who gets accepted, and
who gets rejected? Part of the answer is going to be found in the strength of
your proposal.

This is why it's so crucial that you start interacting with us long before
the deadline. Send us drafts, and we will send you feedback. Edit, then
resend. Repeat. The best proposals we have are on draft 3 or 4 or 5. They
always get better over time. Always.

If we have to make a tough decision between two projects, one of which is a
lousy proposal and the other one is a great proposal which we've been giving
feedback to over a period of several days, the choice becomes pretty clear.

Proposal iterations demonstrate your understanding of the topic at hand (it
takes understanding to revise a timeline, etc), demonstrate your willingness
and ability to interact with the community, and demonstrate that you are
taking GSoC seriously. These are all good things for potential mentors to see.

## Pick a Good Project

This one should be obvious, but judging by some of the proposals we've
received it may not be. You need to pick a good project. "Good" in this
sense means a few different things. As I mentioned above, it needs to be
beneficial to Parrot. We won't agree to mentor and fund just any project, it
needs to be good for us. Don't propose a project written in a language that
doesn't run on Parrot. Don't propose a project which isn't useful to the
Parrot community at large. Don't propose to write software we don't want,
using toolchains we don't support, or to benefit people who aren't us. Don't
do that.

If you put in a proposal that violates any of these maxims, it becomes
pretty clear pretty quickly that you aren't familar enough with our project.
If it's still early in the process maybe we can help point you in a direction
that would be more acceptable. If you put in your proposal at the last minute
before the proposal deadline, you might be trapped. I always recommend that
you do not wait until the last possible moment. Of course, every year there
are students who do push it down to the wire. These students are at an
immediate disadvantage, a disadvantage that would be extremely difficult to
overcome.

Of course, Parrot isn't the only party here. You need to pick something that's
going to be useful and interesting to you. This has to be something that
you're going to get excited about, because you're going to be spending several
hours working on it every day.

Looking at a list of ideas on a webpage is nice. You can find an idea that
interests you, and base a proposal around that. However, this shouldn't be
your only option. Feel free to propose something that's not on the list. Feel
free to take the kernel of an idea from the list and turn it into something
that works for you.

Don't just pick the easiest thing on the list and expect to breeze through
GSoC. That doesn't do anybody any good. You're going to be working on this for
three months. If we have the choice between a student with a small and easy
project, or a student with a hard and ambitious project, we will try to get
the most benefit for the same amount of time. Don't offer us the moon if you
can't deliver it, but don't think that we are going to be impressed by small,
safe projects either.

By the same token, the project you pick is a reflection on you. If you pick a
small, easy project but have a background to suggest you have more than enough
skill and experience for a larger one, it suggests to us that you might not be
a hard-working or competent coder. Or maybe you are competent but lazy and
looking for an easy paycheck. There are a million possible combinations here,
and none of them reflect well on you as a candidate. We do list easy projects
in our page of project ideas, and we do mention them because they are things
that we want done. Some younger students need easier projects to help them
get involved, and to make up for all the experience and theoretical knowledge
which they are lacking. Also, most easy projects can be expanded to become
harder and more ambitious. My point is that we can sniff out people who are
trying to coast through the program, and we don't like the smell of it.

Most importantly, once you have an idea in mind, *talk to us about it*.
There's no escaping it, at some point we are going to need to chat. Talk to
us about your idea to get feedback about it. We can help you focus on the
right issues, and refine your good idea into a great one. We can help you fill
in some of the important details, and we can make sure you find a project that
is going to maximize benefit to both parties.

Of the students who came to us early with ideas, not a single one of them has
kept their ideas unmodified. In all cases, ideas have gotten *better* with
feedback and revision, and better ideas are more attractive to us.

## Slots and Proposals

At the time I am writing this, I don't know how many slots Google is going to
allocate to Parrot. Depending on the number we receive, the process of picking
students could be relatively straightforward or down-right painful. I honestly
don't know which way it is going to go.

Even if we get lucky and Google provides us with a huge surplus of slots,
there are several proposals that we will not accept anyway. In some cases,
students have proposed more than one proposal and we can only accept one
proposal per student. In other cases, several students have proposed the same
project (or similar projects with very small variations between them). We
probably won't accept more than one proposal on the same project. In the first
case of the student with multiple proposals, this is a good idea. It helps to
hedge your bets, if you have enough time and energy to maintain and iterate
on multiple proposals at once. In the second case of the single project with
multiple students, this is a very big problem. Regardless of the quality of
the proposals, multiples will probably not be accepted. Students who come talk
to us ahead of time can be warned about conflicts. Students who do not could
be blindly walking into certain failure.

Can I say it one more time? Come talk to us. The earlier the better. If you
put in a proposal with no feedback from us and it turns out to not be
acceptable, you probably won't be able to be accepted.

Parrot will not just fill up all the slots that Google gives us if we don't
have enough good proposals. We will turn down bad proposals and return unused
slots to Google. Remember that GSoC requires an investment of time and energy
from our developers, and we won't waste our resources on bad proposals. We
would rather have fewer good proposals than to have many bad ones.

We currently have 15 proposals for consideration. I am very confident that
at least 6 of them, in their current condition, will not be accepted no matter
how many slots we receive. We have over a week before a final decision must be
made, so it's very possible that some of these weaker proposals could be
turned around in time. We have about 4 or 5 proposals which are absolutely
spectacular. It is unlikely that these get knocked out of the ranking over
the course of the next week, although anything is possible.

## Next Year

This has been a short, opinionated treatise about making successful GSoC
proposals, with an especial focus on the Parrot Foundation. I hope it can help
some students next year.

Even though the application deadline is passed, we're not without hope quite
yet. Now we're in a period of feedback and interaction where we have to pick
successful applications from our list of submissions. If you  have submitted
a proposal late and didn't take some or all of this advice, you still have
time to do it. Don't wait! The clock is ticking, and we have only a very
limited amount of time to do everything I say above. Take this seriously
and treat it like a job. It's important and the benefits are real.
