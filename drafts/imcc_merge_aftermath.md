---
layout: post
categories: [Parrot, IMCC, GC]
title: IMCC Branch Merge Aftermath
---

As I mentioned a few days ago, the `imcc_compreg_pmc` branch had gotten merged
into trunk. For a while, all seemed well and we were in good shape for the
3.3 release of Parrot on Tuesday. This turned out to all be an illusion.

Parrot and Rakudo hacker Moritz came into the IRC channel to report that he was
seeing segfaults randomly during the Rakudo build and during the Rakudo spectest
suite. The segfaults (and, as we found later, assertion failures) had a number
of interesting properties: They were not easily reproducible, they manifested
in different ways, in different places, when run by different people on
different machines (and, in some cases, by multiple users on a single machine).
All these properties pointed to the underlying cause: Parrots GC.

What makes debugging GC bugs so hard is that they tend to be highly influenced
by memory layout and personal settings. Different GC threshold values and
memory limits will cause GC to run at different times. Depending on when GC
runs, and what it collects with each run, the bugs will appear differently. Or
if GC doesn't run in the "sweet spot", the bugs might not appear at all! What
appeared to be the saving grace here was that the GC core used didn't seem
to play an effect. Some people were using the new GMS core, I was using the MS2
core. We were all seeing problems, which suggests that the bug was external to
GC. This is especially good if you don't want to spend your next several
afternoons trying to debug a strange corner-case error inside a GC.

## The Problem and Early Fixes

At first, Moritz was seeing bugs during the Rakudo spectest. plobsing and myself
were seeing similar-looking segfaults during the Rakudo build. Specifically, we
were seeing problems where PMCs located in the constants segment of a PackFile
were being prematurely collected. I committed a quick hack/fix to prevent GC
from running during packfile serialization, and plobsing committed a fix to
force the serialization mechanisms to properly mark temporary variables. With
these two fixes in place, I was no longer seeing *any* bugs. I could build
Rakudo and run all the tests and spectests without seeing any problems. Moritz,
in contrast, could now consistently build Rakudo without problems, but saw many
segfaults during spectest.

Moritz, as I mentioned before, was using the GMS GC core by default. I was using
the MS2 core, since that is the default option with Parrot and the one we need
to test with until we replace it with GMS after the 3.3 release. It's impossible
to prove the negative "the bugs do not ever happen with MS2", it was starting
to look like the remaining bugs were GMS-specific. 

On one hand, debugging problems with a single GC core is a pain in the ass. On
the other hand, the GMS core adds a new mechanism called write barriers which
MS2 did not have. There were a few facts that lead me to believe write barriers
were the problem:

1. Through simple bisecting we could prove that the bugs definitely appeared
   after the `imcc_compreg_pmc` merge. That branch did not touch GC internals at
   all.
2. The GMS core was working fine building Rakudo and running its tests prior to
   the merge. GMS was also well-tested under other types of workloads. 
3. The `imcc_compreg_pmc` branch did create a new mechanism for storing
   packfiles in PMCs, but didn't add any new write barriers.
   
These facts together seemed to point to the idea that we were not using write
barriers correctly after the branch merge, which was leading to premature
collection of PMCs and eventually the segfaulty behavior we were seeing.

## Write Barriers

So what are write barriers? GMS is a generational GC system. Generational
systems break objects into generations depending on how old they are. The theory
is that older objects are old precisely because they don't need to be collected.
And if they don't need to be collected, it's a huge waste to spend the extra
time marking them. This is how generational GC schemes save time and effort:
they separate out objects into piles of them which are "unlikely to be dead"
from younger, fresher, "more likely to be dead" objects. Then we can avoid
marking the ones which are not likely to need it.

However, this raises something of a problem. What happens if we have objects
from different generations referencing each other. Let's say we have two objects
OLD and NEW. OLD is an older object in an older generation which is long-lived
and is unlikely to get collected. NEW is a fresh new object which is in a newer
generation. OLD contains a reference to NEW.

    OLD -> NEW

Now we run a GC mark and sweep cycle. OLD is in an older generation so we don't
mark it. If we don't mark it, we don't follow all of its references. If we don't
follow all the references, we don't mark NEW. OLD is fine since it's in an older
generation and it doesn't get swept. NEW, however, does. 

    OLD -> Invalid Pointer

The next time we try to use NEW, we instead are using an invalid pointer, and
that causes a segfault (or, in a few cases, assertion failure, weird exceptions,
etc). The solution to this problem is to use a *write barrier*. Write barriers
tell the GC that the older object has younger children. We need to add the old
object to the list of objects to be marked during the next mark cycle to make
sure we don't lose the newer child objects.

## The Current Problem

Now that we know what write barriers are, why are they a problem after the new
`imcc_compreg_pmc` branch merged? In the old system, IMCC would always return
a `PackFile*` structure pointer after a successful compilation. This packfile
contains, among other things, an assortment of PMCs: constants, subroutine and
method objects, call signature arrays, etc. These PMCs are technically constant
objects, but still need to be marked to save them from GC. In the old system,
packfiles were automatically merged into the single global packfile and were
always marked from there.

In the new system, we try to improve encapsulation. IMCC no longer has free
reign to fiddle with global data, such as the single global packfile instance
in the interpreter. Now, IMCC returns a packfile PMC, and the interpreter can
choose whether to merge that packfile in to an existing one, or write it out
to a separate file, or whatever. This raises a problem, because now all
packfiles aren't automatically included in the GC mark. To get around this and
temporarily preserve the old behavior, I add all new packfile PMCs to a global
registry of PMCs which always get marked. These PMCs never die and are always
included in the mark.

When you hear the words "always get marked", that really becomes analogous to
"in the oldest generation". The packfile PMCs end up in the older generation
because they are obviously long-lived and never get collected.

Here's where the old system and the new system clash: The whole old packfile
system relies on `PackFile*` structures throughout, not PMCs. In fact, the
system makes no allowances for PMCs at all, and there are no opportunities to
insert write barriers in any of the API functions. When the contents of the
`PackFile*` inevitably change, the parent PMC object isn't write-barriered and
then the hilarity begins.

## Next Steps

bacek has put together a new fix branch to make PMC wrappers for packfiles more
ubiquitous, and therefore make writebarriering possible in all common packfile
operations. Unfortunately, that branch is not quite mergeable and might not be
ready before the release. If it cannot be made to work, I am going to unmerge
the `imcc_compreg_pmc` changes from master so we can cut a clean release. After
the release we can bring all the changes back in along with bacek's new branch
to get everything working perfectly again.

Unfortunately our hands are tied with the release on Tuesday. We just plain
don't have enough time to really explore before we have to cut a stable release.
We also can't cut a release which is not stable, no matter how tempting it might
be from the standpoint of time and energy expenditure.

Plan A is to get bacek's changes working properly, but our time line for that
is running short. Plan B is to unmerge the `imcc_compreg_pmc` branch and related
changes to trunk. I'm going to get started on that now as a precaution.
