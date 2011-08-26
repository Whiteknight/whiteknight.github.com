GSoC student benabik didn't quite finish his GSoC project this year. For
those of you not keeping extensive notes, benabik's project was two-fold: to
rewrite PCT, the Parrot Compiler Toolkit, in NQP and to add the ability to
generate .pbc bytecode files directly from the parse tree without having to
output and separately compile PIR code first. The first part of the project
went well enough but took up far too much time. NQP-rx, the flavor of NQP we
use most of the time in Parrot, is written itself using PCT, so benabik's
work this summer involved lots of (very painful) bootstrapping and circular
logic. The second part of the project went better than we expected as well.
He is able to generate bytecode, but only for small programs. At the moment I
am not certain whether the limitations he's seeing are inherent in Parrot or
are in the new pbc generators. In either case, we would like to fix them.

But that's not the entire story. See his [final blog post][benabik_post] of
the summer for more details. It seems like the new version written in NQP-rx
is *two times slower* than the old version written in PIR for the same tasks.
That's an enormous slowdown, and really means that we might not be able to
merge the work at all. Once Parrot updates to 6model, and we can migrate to
the new NQP with its various optimizations and maybe reclaim some of that
speed. Until then, this work is sort of in limbo.

[benabik_post]: http://www.parrot.org/content/end-gsoc-not-time-stop...

Of course, benabik didn't just translate some code, and stitch together some
modules, and run into bad results. He learned a heck of a lot as well, and has
started planning to make the situation better. Sometimes, it's the
non-tangibles that are the most valuable results of the GSoC summer.

He's talking about, and I fully agree with him, writing up a new compiler
toolset separate from PCT. The new toolkit may even become a PCT replacement
in the long run. He had posted some of his ideas for that on his blog. I'm
going to reiterate some of them, and talk about some of my own ideas for such
a project. He's calling his planned rewrite the Parrot Alternate Compiler
Toolkit (PACT), which I like.

First thing, benabik wants to write PACT in Winxed instead of NQP. When he
started the project at the beginning of the summer Winxed was much less
mature and usable than it currently is. It's only one or two features away
from being able to do a direct line-for-line translation from NQP to Winxed,
and those things could be added pretty quick. Winxed is, and always has been,
much closer to the Parrot VM than NQP has been. Winxed has always been driven
by the need to add nice, readable syntax to Parrot features. NQP on the other
hand has been driven in large part (though this is not the only motiviating
factor) by Perl6 syntax and semantics. NQP has been an excellent language for
bootstrapping the Rakudo Perl6 compiler, and it is only getting better in that
regard over time. However, it hasn't always been the best choice for writing
lower-level Parrot system utilities and libraries. However, it is precisely
this niche where Winxed shines. Winxed really is designed for low-level,
semantic-light, HLL-agnostic library and utility work, and that's why it would
seem to make a great alternative for this sort of library.

A new compiler toolkit should really be low level. It should be the bottom of
the abstraction layer, as far as an HLL compiler is concerned. It should
provide tools, but leave plenty of options open for subclassing and
customization on a per-case basis. The focus of it, from the code to the
documentation and examples, should all be on customizability. In the world of
HLL language compilers, there is no one-size-fits-all solution, and we need
to embrace that.

We have the tree-optimization library which was added as part of GSoC last
year. It has been an optional add-on to PCT for some time, but hasn't really
gotten any love. With PACT, we would like to rewrite tree-optimization, and
deeply integrate it with the rest of the routines. Optimization functionality,
if not any individual type of optimization, should be available by default and
as easy to use as possible. We should be able to have a library of ready-made
optimizations, and the ability to add new optimizations easily.

This year, tcurtis has been working on an LALR parser for Parrot called
Lalrskate. His project hasn't been completed either, but a lot of great work
has been done. If we can take his frontend and modify it so it generates PACT
parse trees, we will have a very powerful additional tool for the toolkit.
Likewise, if we can modify plobsing's Ohm-Eta parser to generate PACT parse
trees, we will have that option available as well.

We have lots of little things like this spread out across time and space. By
taking good ideas, bringing them together on a standard interface and
providing them to the user for free, we can make HLL compiler creation on
Parrot even easier than it currently is. Then, if we can take our NQP compiler
and have it also generate PACT trees instead of PCT trees, suddenly it can
start to leverage all the new features as well.

benabik also talks about a lot of the technical details of PACT. Compiler
stages should be objects with state and full APIs, not simple functions. We
should use a Visitor pattern to traverse parse trees, instead of the custom
approach used now. Using Visitors for all stages, and having a common
interface for that suddenly makes things like tree-optimization trivial. We
can also make some better design decisions, like better encapsulation for the
register allocator, better separation of the code emitters, more
interchangability between various components, easier customizability, etc.
We should be able to easily plug in new frontends like Ohm-Etal, Lalrskate,
and others (like rohit_nsit08's corellascript bootstrapped Javascript
compiler) to easily generate PACT parse trees, and then have multiple possible
output vectors like PIR, PBC, M0, other higher-level languages, maybe
syntax highlighters, documentation formatters, etc. Maybe even code analysis
tools as well.

If we take the stuff we have and put it under a common umbrella and use good
clean interfaces for extensions and modifications, it should become much
easier for future projects to plug directly into it.

I'm excited about this idea. I've wanted to start adding more
compiler-building tools to Parrot for a long time now, and this seems to be
the most promising path forward. Rewriting PCT in NQP or anything better than
PIR was a great first start, and starts to make some of the logic more
accessible. Knowing what we know about it now, we can start putting that
knowledge to good use to build the system we all really want.




