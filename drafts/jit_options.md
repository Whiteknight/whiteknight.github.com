JIT seems to be one of those topics that we can't seem to stop talking about.
Every few days it seems like we mention it a little bit, if only just in
passing. Everybody's favorite bacek has even been writing some pretty
impressive-looking code to access the JIT portions of LLVM using NQP and NCI,
which is more interesting perhaps as an exercise than as an actual
"production" JIT solution. Something is always better than nothing, and I
think everybody is waiting on pins and needles to see what he comes up with.

Lorito is supposed to be this big enabler, an important and necessary stepping
stone to get us into position to finally implement a JIT. We don't have Lorito
yet, not even really a prototype as I am writing this, although things can and
likely will progress quickly once the powers that be decide to put finger to
keyboard with it. Without Lorito we really can't do more than plan and
conjecture about the new JIT system, but we also don't want Lorito to arrive
on the scene without having a firm plan in place for how to proceed.

The current plan is to use LLVM's JIT functionality as the underlying code
generation mechanism for Parrot's JIT. This plan has the benefits that we have
it written down on our road map already, that several of our developers are
at least passingly familiar with LLVM already, that bacek's prototype code
uses LLVM to implement a working toy JIT, and that LLVM does seem to support
all our target platforms. Considering we are embarking on what could become
a huge project, and are going to devote large amounts of man-hours to it,
these points of familiarity and convenience are not able to be overlooked.

Whenever we raise the topic, however, there are always people around with dire
outlooks about it. "LLVM isn't a good JIT for dynamic languages", they say.
"Just ask the Rubinius/PyPy/UnladenSwallow people". Saying that something is
bad or inferior without offering up an improved alternative really doesn't
carry a hell of a lot of weight with me. After all, I throw around criticisms
and condemnations of various bits of software all the time, but the ground
does not shake and the code does not immediately change because of it. No plan
might be better than a terrible plan, but a decent (if not great) plan
certainly must be better than both. In other words, if we can have *any* JIT
which both improves general Parrot performance and doesn't significantly
decrease maintainability or create other problems, that has to be better than
having no JIT at all. If we could have a system which was flexible and
pluggable enough to replace LLVM with something else in the future without
having to rip Parrot to shreds, all the better. I won't get my hopes up too
high about this last part, however.

If we don't use LLVM for our JIT, and I can't say I am inseparably married to
the idea after all, what are our alternatives?

GNU Lightning appears to be extremely sluggish in terms of development, if
it's even maintained at all. I've heard more than one rumor that it was
essentially frozen or even abandoned, although I can't seem to verify those
rumors myself. I can tell that the webpages I've looked at today for GNU
Lightning are not substantially different from how they were about a year ago
when I last looked. The Wikipedia article for it doesn't appear to have been
substantially edited since I last looked, although that's not necessarily a
definitive resource on the subject. GNU Lightning also doesn't seem to have a
lot of features we would probably want in a JIT, like register allocation and
optimizations.

JIT can be a win in several ways: By using type specialization information
gathered at runtime to replace expensive dynamic dispatch with more direct
variants, by inlining op code bodies and maybe more code as well to reduce
dispatch and call overhead, by optimizing code which is run frequently, in the
hopes that the overhead of optimization is outweighed by the runtime savings
of "hot" code snippets, etc. If GNU lightning doesn't provide optimizations
or automatic inlining (which is typically an optimizatin), what good does it
do us? There's only so far we are able to go with type-specialization alone.
We could manually do inlining on the Parrot end, but that would probably
require some evil dark magic or brain-dead code duplication, something that we
want to avoid.

Similarly, I am having a lot of trouble verifying the livelihood of libJIT,
another JIT library which we have looked at in the past. I would love to find
some resources about this, but I just can't say that I am seeing enough
evidence today that the libJIT project is alive and well. And being "alive and
well" in April 2011 isn't really as good as saying that our eventual JIT
solution will continue to be alive, vibrant, and under active development in
the next few months when we actually start work on our JIT solution, or years
later when we are having to modify and maintain it. We really need a JIT
project which is active today, active tomorrow, and active in 10 years when
Parrot 13.3 is released (to much acclaim and fanfare, I expect!). We need a
JIT solution that isn't being kept on life support by a single dedicated coder
with a google code account and a low-volume mailing list. Again, I don't know
what the status of libJIT is, but a cursory search of the Internet doesn't
seem to turn up much promising recent information.

nanoJIT is another tool that we are becoming more interested in. A year ago
when I first looked at it, what I saw wasn't too promising. The nanoJIT code
base was in the middle of a huge merge, and there were a few other problems
related to the immaturity of the codebase that worried me. I can't remember
all the details because I didn't take detailed notes, but I remember being
both unconvinced and uncompelled to look at it any further. However, nanoJIT
does not appear to have sat idly by the way some other JIT engines appear to
have done. If anything, nanoJIT has become much more impressive over time and
now that they seem to have most of their major community issues sorted out it
might be a much more interesting alternative. What is most interesting about
nanoJIT is that it's designed from the ground-up to support a tracing JIT,
which is typically considered to be one of the better algorithms for it. It's
described as being small and fast, and also seems to have at least some
support for all the major desktop architectures that Parrot calls home. If it
wasn't before, nanoJIT is definitely an option worth looking at in more detail
now.

One thing I do worry about with NanoJIT is that it's always going to be
obeying a higher master: Tamarin and FireFox. nanoJIT was designed to run
JavaScript code in the setting of a browser, and where the browser needs the
JIT to change those changes will happen. Does Parrot want to be at the mercy
of a dependency which changes whenever the primary user requires it to change?
I may not understand the details of the relationship between nanoJIT/Tamarin
and FireFox, but there's a synergy there that either we cannot come between
or we would not want to anyway.

The next option maybe is to roll our own, and write our own code generator.
Besides the fact that we have been down that path already with results best
described as "uninspiring", there are a number of problems with this approach.
First, it becomes a huge issue to develop the architecture and implement all
the code generating back-ends for the various platforms we support. For many
such platforms, like PPC and SPARC, the number of people using Parrot on those
platforms far outnumbers the core developers who develop on them. I, for
instance, do not use PPC or SPARC, and have absolutely no access to either if
I wanted to start. Not being able to adequately develop and test on those
platforms is a prohibitive problem for us.

The effort to write the code generator is going to be a huge undertaking in
itself. Maintaining it over the long term is sure to be no picnic either. This
of course ignores the fact that we're going to need register allocation done
in a way which is completely different on different architectures,
optimizations which are sure to be platform dependent in part, and other
aspects of a "good" JIT that we don't want to spend the time or the effort
to make ourself. We *really* do want to avoid writing a JIT ourselves, and we
would like to try as hard as possible to reuse an existing solution. The
problem is that so many other dynamic language VMs appear to be "giving up" in
this same search and deciding to implement their own solutions. Sure there are
immediate improvements which could be made through radical specialization on
a single platform or language, but those things don't do anything to benefit
the greater good, and are unusable (and undesirable) by Parrot.

This post was only intended to be a quick recap of some issues related to JIT
and some of the alternatives that we have available to us. If nothing else,
this post should help to demonstrate how much *I don't know* about existing
JIT solutions, and how much more there is to be researched. At the moment
our plan is to push forward with LLVM as our JIT engine. While that isn't
written in stone, it's not a plan we're going to just abandon on a whim
either. Where compelling alternatives exist they will be given fair
consideration, but we need a lot more information before we can even evaluate
what is and is not "compelling".

