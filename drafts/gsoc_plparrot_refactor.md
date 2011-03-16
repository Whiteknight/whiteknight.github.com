As part of my series of GSoC project ideas for 2011, here is an idea about
fixing and refactoring PL/Parrot.

## Problem Statement

> PL/Parrot embeds Parrot Virtual Machine into PostgreSQL. It was written
> before the new Parrot embed API was written, which has a much nicer
> interface. This project will consist of refactoring the PL/Parrot codebase
> to use the new API. In the process of converting to the new embed API, many
> bugs in PL/Parrot will be fixed, since PL/Parrot inherits many bugs from the
> old embed API.

In short, update PL/Parrot to use the new embedding API.

## Difficulty

On the GSoC tasklist page, this task is ranked 4/5. I tend to think that this
project could be on the easier side. I would instead rank it around 1/5 or 2/5
assuming you don't run into any crazy problems which need major fixing at the
Parrot level.

## Deliverables

What we want to see is a working PL/Parrot which exclusively uses Parrot
through the new embedding API. In addition to the working code, you're going
to need to provide tests for any functionality added or changed and
documentation where needed.

In an ideal situation you'll be able to reimplement PL/Parrot using the new
API without changing any of the behaviors of PL/Parrot or changing any of
it's behaviors or external interfaces. If you keep things in that regard
the same or similar, your need to write additional tests or documentation will
be low.

## How to Get Started

If you're interested in working on this project, I suggest you get yourself
copies of Parrot and PL/Parrot to play with. Look through Parrot's new
embedding API, and start figuring out how the functions it provides map to the
routines used by PL/Parrot.

Get in touch with myself and/or dukeleto as well to show that you are
interested in this project.

## Who Should Apply

This project is going to require C. In fact, it's probably all C. You're
probably going to want to know about Postgres as well, although if you know
a little bit about other databases and are willing to learn you might do
well anyway.

If you're thinking about doing this project you should be aware that it isn't
self-contained. The existing PL/Parrot has some bugs, and has some workarounds
for bugs in Parrot. The new embedding API is also not complete in that it
doesn't provide all the routines PL/Parrot is going to need. New embedding
interface routines can be added to Parrot, but exactly how much is going to
need to be added will depend on a lot of factors, not all of which may be
obvious at the beginning.
