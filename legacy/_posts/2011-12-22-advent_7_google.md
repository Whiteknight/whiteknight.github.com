---
layout: post
categories: [Parrot, Advent2011]
title: Advent 7 - Google GCI and GSoC
---

When you want to find something on the internet, Google is a pretty popular
tool to use. When you want to get some code written, Google turns out to also
be a pretty good idea.

Google has been doing it's Summer Of Code (GSOC) program for several years now
and every summer it's a smashing success. Last year they started up with a new
program called the Google Code In (GCI). That program, while wildly different
from GSOC, is also incredibly awesome. Every year Parrot receives a huge
amount of code from these programs, and we would not be where we are today
without them.

The Summer of Code program is very straight forward: Every year we receive
applications from college age and some highschool-aged students for projects.
Over the course of the summer the accepted applicants work diligently and at
the end, if they are successful, they get paid. Also there are cool T-shirts.
The GCI program is aimed at younger students, mostly those in high school.
Instead of one large project spanning several months, the GCI students work on
a large number of bite-sized tasks from many organizations. Each task is worth
points, and the students who have the most points at the end of the program
receive a prize. Also, I think there are monetary rewards for completing a
certain number of tasks.

Last year we had a huge number of tasks completed by several GCI students. We
had many bits of code written or re-written, some new documentation written,
and many many tests added. Our project-wide test coverage metrics increased
dramatically, since we had several tasks devoted to test coverage and several
talented young coders who were chasing them down. Once you figure out how to
write tests, subsequent tasks go much more quickly. We had some students who,
after getting the hang of things, were able to complete several tasks per day
at the end.

I heard one or two accusations that Parrot was inflating point values by
offering tasks that could be completed so quickly. I pointed out that many
tasks were considered very hard by students at the beginning, but as their
familiarity of the code increased the relative difficulty decreased. It wasn't
a problem of Parrot's tasks being too easy, but the students learning and
improving much faster than we could keep up with. By the end of the program
we were learning that the difficulty really needed to ramp up over the course
of GCI, because students would be capable of much more towards the end than
they would be at the beginning.

This year things are going a little bit more slowly. We have fewer of the
"increase test coverage" tasks, because our test coverage is still so high
from last year in the core VM. I've scoured several potential sources of tasks
including searching for TODO notes in the Parrot source code, scanning through
our myriad of trac tickets, digging into ecosystem projects (Plumage and
Rosella especially) and begging other contributors for task ideas. Already
we've seen some very useful bits of code contributed to the project, including
new features and impressive cleanups and refactorings of old crufty code.

GCI this year is essentially closed off. There were two opportunities to
publish new tasks and those have both passed. Students are slowly working
their way through the remaining tasks in the queue until the program ends in
a few weeks. However, next year I'm sure we're going to see both another GSOC
and another round of GCI (At least, I hope we see them, since these are both
so awesome!).

Despite being months away we're already starting to look for project ideas and
potential applicants for GSOC. Getting good ideas picked early allows us time
to refine them, and getting familiar with potential applications now helps us
to learn more about them and be more comfortable with them when time comes to
make final selections. GCI doesn't require so much preparatory work, but it
would be nice if we went into next year with a larger pile of varied tasks for
students to work on, instead of needing to scramble to create tasks in
sufficient numbers at the last minute.

GSOC and GCI have both been amazingly successful programs in the past and I am
hoping that trend continues into the future. Parrot has benefited from both
programs to an amazing degree, and with a little bit of luck and a lot of
planning we can keep the train moving into 2012 and beyond.

If you're a highschool or college-aged coder, or know somebody who is, and
would like to talk about getting involved with either program in 2012
(especially if you would like to work with Parrot specifically!) please let
me know and I can make sure you get pointed in the right direction.
