---
layout: post
categories: [Parrot, GSOC]
title: GSoC Idea - Parrot Debugger
---

## Problem Statement

> Parrot needs a new debugger. There exists a toolset called
> "[parrot-instrument][]" which can be used to insert introspection, analysis,
> and manipulation functions into Parrot internals. You may optionally
> (preferrably) use Parrot-Instrument as the basis for your new debugger or
> create a new one from scratch. A successful debugger will have the ability
> to set break-points on subroutines at the PIR and HLL level, set watchpoints
> on registers or individual PMCs, and introspect state information from a
> running program. The debugger should read information from annotations so
> that it can be used to work with various HLLs. Performance of code executing
> under the debugger is not a primary concern for the initial deliverable.

The short version is this: Make a *good* debugger for Parrot. Our old
debugger is old and broken, and didn't implement enough features. Our new
debugger should be better designed, so that we can maintain it and extend it
in the future.

## Difficulty

*Difficulty*: 4/5

The difficulty level listed for this one on the Parrot wiki is 4/5, but it
could be lower than that depending on how you frame the proposal. A basic
debugger should have the ability to set breakpoints by function name and/or
by file and line number. You should be able to step into line-by-line, step
over line-by-line, and run the program. You'll also want the ability to print
backtraces and also inspect the values of PMCs and registers.

Those are the features needed for a basic debugger, but not necessarily
everything that would need to be included in a GSoC project. If your debugger
did half as much but was well designed, well written, and easy for other
people to hack on after the summer was over, that would be a good thing.

## Deliverables

For this project you're going to need to make a debugger, but also all the
supporting details too: A build system, unit tests, documentation, and
examples. When you're done, the debugger should be easy for other people to
hack on to maintain it and add new features.

## How to Get Started

Parrot-Instrument is currently broken, but not too badly. If people are
interested in this project, we can try to fix it for you (and you can help
if you want!). Get in touch with community members on IRC and on the mailing
list. Two people in particular who would be good to talk to about this project
are cotto and myself.

## Who Should Apply

Debuggers tend to have some very standard features, but it's not always
obvious what they can do, or are expected to do, if you haven't used
debuggers before. There are not a lot of hard and fast requirements for this
project. Here are some general ideas about how to be qualified for this:

1. You should be familiar with debuggers and how they work.
2. Depending on how you choose to implement the debugger, you should be
   familiar with suitable programming languages: C, PIR, NQP, Winxed, etc.
3. Design is important, because other people are going to be maintaining
   and extending this project after GSoC is over. People will be using this
   on a daily basis. You should be familiar with principles of good design.
4. You should want to make life easier for your fellow programmers.

If this sounds like you, get in touch with cotto or myself and start digging
in!
