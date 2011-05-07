I've written a few posts about threading recently, because that's the topic
that has been on my mind. However, just because I've written a lot about it
doesn't mean that it's a high priority right now. It isn't, at least not for
the community at large. We do have some developers keen to tackle that topic
however, and so it might start getting done soon anyway. We have several
other priorities that we as a community really want. These are typically
things that our users want and which are developers are committed to deliver.
Today I'm going to list out what I think are some of our priorities to tackle
between now and the 4.0 release. If all these things go well, I think we are
in good shape and we can start moving on to the next level of cool stuff to
talk about.

## Packfiles

We really need to improve our packfiles situation. We've done a lot of work
recently in preparation for changes that we need to make. The IMCC work was
satisfying by itself, but one of the biggest motivations for it was to work
towards proper encapsulation of the packfiles system so we can start
refactoring it.

So what exactly do we need to do for packfiles?

First off, we would like to have them be properly GCable. Currently all
packfile-related PMCs are permanently registered with the GC so they never
get collected even when they are no longer needed. This is wasteful. What
we really need is a system where we can detect unneeded packfiles and recycle
them back to the memory pool. We now have the ability to use PMCs in most
places where packfiles are needed throughout the system, which is a big step
in this direction. The next step is to make sure those PMCs are the correct
type, that we are storing the correct data in the correct places, and that we
can use them the way that they are needed.

We also need to add some new features to allow partial serializations and
other things that Rakudo needs. I'm not even sure I understand all the changes
Rakudo is asking for, but you can be certain I'm going to start my research
on the topic ASAP.

## The Object Model

We have new JavaScript and Python compilers in the pipeline as part of GSoC.
They are going to need new object models, because they are extremely bad fits
with our current object model. In addition, we have the Cardinal (Ruby)
compiler which is (slowly) moving to 6model as well.

I've said before that we want 6model in Parrot. It's a high priority for me
personally, although I can't speak for everybody. We know Rakudo is going to
be using 6model sooner than later. We know Python and JavaScript need it for
performance reasons (among other things). We know Cardinal wants it. Actually,
let me go backwards. We know Python and JavaScript need *something* besides
what Parrot offers now. That something doesn't need to be 6model, but I am
highly convinced that it is one of the better solutions available right now.
6model will certainly be a much better first implementation for these two
projects than Parrots current offering.

Anyway, several important compiler projects either need, want, or could
benefit from 6model. Combine that with the fact that our current system is
an abomination. It's a terrible hack of bad ideas and quick fixes layered on
bad design and prototype implementations. It's time we graduated to something
better.

I would like to start the work of migrating Parrot to 6model sooner rather
than later. If we can start work on it sometime this summer to work in
parallel with the Python, JavaScript and Cardinal work, that would be the
best.

## Compilers

As I mentioned before, we have JavaScript and Python compilers in the works
for GSoC. Having *good* compilers for these two languages will go a long way
towards raising the profile of the Parrot project. We want these two compilers
in a big way. We also want to help the Cardinal project succeed, Ruby is
another popular language that needs a home on Parrot.

After those things, if we can get a decent PHP compiler started, we will be
in business.

Compilers are not things that make the Parrot executable better, faster, or
whatever. What they do is attract more attention, raise the profile of the
project, and start to expose some of the awesomeness of Parrot
(interoperability being a big one).

Sometimes we need to improve Parrot itself. Other times we need to help bring
success to users working on Parrot.

## Lorito
