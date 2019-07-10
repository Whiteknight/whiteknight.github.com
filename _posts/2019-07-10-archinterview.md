---
layout: post
categories: [JobHunt]
title: Interviewing Architects
---

We've been back to interviewing for some architect-level positions again, and I thought I would share my approach to that process, what we look for, and what are the red flags we try to avoid. I generally like to do three phases of the interview: free questions, a code sample and a whiteboard design question. The whiteboard bit is the meat of the process and my thoughts on that are long enough that I will separate it into a second post. Today I'm going to give an overview of our interview process for Architect-level candidates.

## What We Want

I've learned that every company seems to see an "architect" differently. Some places want an "ivory tower" type, who predominantly works in documents and diagrams, and rarely works on actual code. Some places look for somebody who is basically just a "super senior developer" to join a team to bring a little bit of extra experience. I see an Architect as somebody with the following qualities:

1. Should have solid understanding of the concepts in a system from end-to-end (data, networking, applications, etc)
1. Are the last line of defence when it comes to tricky questions or difficult problems.
1. Have enough attention to detail to be able to get the hard things right. They should be trusted with the performance-critical, security-critical, foundational shared code that other devs wouldn't necessarily be trusted with
1. Be able to take the high-level view of things and see larger patterns and concepts that individual team members might not see while focusing on just their own work.
1. Be able to teach and mentor, and help improve the quality of work in others.

I would constrast this with somebody who is a Lead (a little more practical, a little less theoretical, with leadership and delegation skills) and a senior dev (autonomous and with some expertise). It's very often at this level that a person interviews for one job but comes across as a much better fit for something else (and it's not a "failure" per se, just a recognition that we all have different strengths and weaknesses, and not every candidate is suited for every role).

(It's also worth noting, though I won't talk about it in this post, that we would like to see a diversity of people on our team, though this is frequently not the case. I find that, by the time we get to the point of the on-site interview the candidates are almost exclusively male and a mixture of white or asian. Exactly why this is, and what can be done to correct it, are interesting and important questions.)

## Pre-Screening

I'm not going to talk too much about the period before the interview, when resumes are reviewed and some candidates are called for a phone screen. Basically, we're not bringing somebody on-site without some expectation that they would be suited for the role. We like candidates who have experience with our tech stack, but we wouldn't exclude a person just because they worked on a different stack.

## A Few Quick Points

In no particular order:

1. **Shorter resumes are better**. Single page is best. Spotlight the keywords and don't make me search for the few relevant bits in 8 pages of edge-to-edge prose.
1. I don't really care what you wear, but some people do. You don't need a full suit, but I would at least stick to a collared shirt and slacks (or, whatever is equivalent for a woman).
1. Knowing the answer is great. Saying "I'll have to research that and get back to you" is good. Bullshitting for 20 minutes in an hour-long interview when you don't know the answer and don't want to admit it is not great.
1. Calm down. You've been doing this stuff for years, you know it inside and out.

## Free Questions

The interview always starts with a short question-and-answer period, mostly driven off the contents of your resume, and then diverging off to explore other topics as they arise. This is a major reason why keeping the resume concise is important. If I can't find the interesting topics to talk about, I won't ask you the good questions and you wan't have an opportunity to showcase what you know. We've passed on candidates who might have been a good fit for this reason, because we didn't know what to ask to give them an opportunity to shine. Two pieces of advice:

1. *If you want to talk about it, put it on your resume*. Show us what makes you a strong and interesting candidate.
1. *If you don't want to talk about it, don't put it on your resume*. It should go without saying, but if we catch you lying on paper, we definitely won't hire you.

The point of this part of the interview is mostly to get a sense of you as a candidate. Can you speak confidently about different areas of software and enterprise design? Is your experience broad or narrow? Are your skills transferrable to our problems? What have you done and what more are you capable of? I would say that the course of this portion will really frame the rest of the interview process, and could either land you in a big hole or have you placed up on a high pedastal.

## Coding Sample

The coding sample we give basically follows 3 rules:

1. We never make you do any coding on paper or on the board. I see very little value in those kinds of exercises
1. We typically let the candidate take the problems home, mull them over at their own pace, and submit when they're ready.
1. The questions tend to be quite "simple" technically

That third point is the most important, we're asking simple problems because we're not testing your knowledge about a specific data structure or a specific algorithm. Instead, we like to see your approach to the problem. Are you organized or disorganized? Do you employ unit testing by default? Is your code style consistent, readable and maintainable? Are you making use of modern language features (and thus are showing some effort to keep your skills relevant) or not?

In theory, we could ask the same exact question of developers at all levels. Junior developers would, we expect, be lucky just to get to an approximate solution without some coaching. More senior-level devs should be able to come up with a good solution, which hits all the major requirements. Architects should be able to produce code which is not only correct but exemplary, something we could show to the junior devs to teach them about best practices. It always surprises me how, even among Architect candidates with 10 years or more of experience, we still see some of these issues:

1. The code comes as just a bare source file, with no project structure, no unit test projects, no harness, no documentation, no sense of organization, nothing.
1. The code has absolutely no structure, no classes, no demonstration of OO design principles at all.
1. The code doesn't produce the correct results, or only works for a very narrow set of test inputs
1. Variables and methods are poorly named and the code is unreadable

Part of the problem we have, and it boggles my mind every time I see it, is that we get people with years of programming experience who can't write a simple program. This is part of the reason why so many companies turned to the [FizzBuzz](http://wiki.c2.com/?FizzBuzzTest): So many people can't do it. Seeing that you either can't solve these problems or can't solve them in a good way is a major red flag and generally disqualifying. Sometimes we can save a candidacy if the candidate has a public Github account or other code portfolio where we can see good examples, but in lieu of that, a sloppy, lazy or crappy code sample is basically the end of the line.

As an aside, I know that people are busy and people are often working a full-time job while they're interviewing, and they can't devote hour after hour to these kinds of problems. That's why we ask simple problems, so it doesn't take a lot of time to understand them and code a solution. Doing some basic code formatting, doing a little bit of refactoring to make things nicely organized, throwing in a unit test or two, and showing a modicum of knowledge of C# or OO design in general is not hard and does not take long.

Basically the question we're trying to answer is, "What kind of code are you going to produce when we hire you?" and if the sample we see is a dismal one, that isn't a great indicator that we should hire you.

After the code sample is submitted we typically like to schedule some time, either on-site or remotely, for a defence. We'll go over your sample, ask you to explain your process and your reasoning, and answer any questions we have about it.

## Overview

Some companies have marathon interviews that last hours, putting the candidate through all sorts of discussions, tests, projects, panels, etc. We tend to keep the process more short and sweet, relying on the few indicators we know correlate well with performance and ignoring the historical cruft of things which don't. We can usually answer in only about an hour whether we want to work with this person or not. So, if you have a chance to get in for an interview, make sure you bring your best effort and be prepared to make a good first impression. There might not be too many other impressions after that.

Next time I'll talk about the part I skipped over here, the whiteboard design question.


