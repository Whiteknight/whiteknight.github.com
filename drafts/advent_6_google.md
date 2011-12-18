---
layout: post
categories: [Parrot, Advent2011]
title: Advent 6 - Google GCI and GSoC
---

When you want to find something on the internet, Google is a pretty popular
tool to use. When you want to get some code written, Google turns out to also
be a pretty good idea.

Google has been doing it's Summer Of Code (GSOC) program for several years now
and every summer it's a smashing success. Last year they started up with a new
program called the Google Code In (GCI). That program, while wildly different
from GSOC, is also incredibly awesome. Every year Parrot receives a huge amount
of code from these programs, and we would not be where we are today without
them.

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

Last year we had a huge number of tasks completed by several students. We had
many bits of code written or re-written, some new documentation written, and
many many tests added. Our project-wide test coverage metrics increased
dramatically, since we had several tasks devoted to test coverage and several
talented young coders who were chasing them down. Once you figure out how to
write tests, subsequent ones go much more quickly.

This year things are going a little bit more slowly. We have fewer of the
"increase test coverage" tasks, because our test coverage is still so high from
last year in the core VM.

One thing we learned last year was that translation tasks aren't really good
for us as a project. We have relatively few people who are both able to review
translations and who are interested in spending the time to do so. Also, Parrot
doesn't yet have built-in i18n support, so translation is essentially
restricted to documentation.

