## Problem Statement

> Create a tool or toolset for calculating code maintainability metrics for
> Parrot. The toolset should operate on PIR or PBC, and be usable for any HLL
> which runs on top of Parrot (using code annotations to get information about
> the HLL). The toolset should be able to calculate total lines of code (in
> the original HLL, in addition to the generated PIR/PBC), per-sub cyclomatic
> complexity (including nested lexical subs), and inter-class coupling.

We would like tools, which operate at either the PIR or PBC level to be able
to analyze and quantify code written in HLLs.

## Difficulty

*Difficulty*: 2/5 - 5/5

This project is sort of open-ended, in that once you have a basic framework
for parsing and understanding PIR or PBC, you should be able to build several
different metrics on top of it. The sky is the limit, and if you start off
with that good foundation it should be possible to do some really great work
before the summer is over.

The first stage is understanding PIR or PBC. In the case of PIR you could
probably piggyback on an existing PIR parser such as PIRATE, and traverse the
existing PAST tree to calculate metrics and things. This is probably the
easier way to go, but it may not be the most flexible since you would be using
an existing compiler and an exising in-memory representation of code.

For the case of PBC you are going to need to write your own routines to
analyze the structure of PBC, and this is non-trivial to do. However once you
have a way to parse out information from PBC, you have a lot more flexibility
for how you store and analyze that information.

## Deliverables

The most basic deliverable is a utility to count the total lines of code. For
this utility, you are either going to need to be able to count lines of PIR,
total instructions in PBC, or be able to infer the number of lines of code
in the original HLL source using annotations. Once this is done, if you have
time left, other metrics can be delivered.

In addition to the count of lines of code and other metrics, You are going to
need to provide all the standard accompaniments: A working build
infrastructure, a comprehensive unit test suite, user documentation, and
working code examples.

### Cyclomatic Complexity

For the cyclomatic complexity metric you are going to need to be able to parse
the PIR/PBC code into a control flow graph object, and then perform
calculations on that graph. Parsing is, as I mentioned above, the biggest
hurdle. Once you have the control flow graph, calculating the cyclomatic
complexity is not difficult.

### Class Coupling

This utility requires you to scan through the code and find explicit
references to classes and namespace. This can be through opcodes such as
`new` or `get_hll_global`, or something similar. You need to count the number
of instances of referencing one namespace from within another namespace, and
then calculate a coupling metric to display that information to the user.

### ...And More

There are plenty of other metrics which could be calculated for software. If
you still have time on your hands, pick one and implement! Some ideas are
function point counts, halstead measures, inheritance depth, instruction path
length (op path length), total number of classes/namespaces, inter-namespace
coupling, etc.

The interesting thing about standardized code metrics is that nobody can agree
on whether they are useful at all. If so, there's no agreement on which
metrics are the best for quantifying software quality or development quality.
With that in mind it's going to be extremely important that we have a
framework capable of supporting multiple different types of metrics, because
different people are going to want to use different ones.

## Alternative Projects

The idea behind this project is the creation of language-agnostic tools to
help analyze and maintain software. If you're already playing around at the
PIR/PBC level, a utility to measure code coverage in a language-agnostic
way would be absolutely awesome on all levels. This could also be accomplished
by using Parrot's existing trace or profiling runcores, and then mapping back
to the individual lines of HLL code.

Other projects that can provide help to programmers in a way that is
independent of the programming language used will definitely be considered.

## How to Get Started

Obviously it's important to read up on the ideas of various code metrics. You
can't be expected to implement a cyclomatic complexity calculator if you don't
know what cyclomatic complexity is, after all.

Code metrics are tools for programmers to help improve what they do.
definitely come to the mailing list and chatroom to talk to various developers
and find out what they need.

## Who Should Apply
