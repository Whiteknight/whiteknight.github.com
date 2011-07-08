I merged in my first packfile branch yesterday, after approval from kid51 and
cotto. There were a few build failures manifesting on certain platforms, but
as of this morning those are fixed. The branch I merged adds in a new PMC type
called PackfileView. PackfileView is the intended replacement for the old Eval
type. It is a thin wrapper around the `PackFile*` structure and provides
several methods which are thin wrappers around packfile subsystem API
functions. With PIR view, PIR users now have access to the same kinds of
functionality as developers at the C level has. It also creates a
reinforcement model for development: The better encapsulated the packfile
subsystem is behind a good API, the more methods we can easily add to
PackfileView, and the more easily we can test it all.

With that branch merged, the death clock countdown for the Eval PMC has
started. It's not something we are going to rush, because there's the
potential for a lot of user code to be affected. But we are moving in this
direction and everybody will benefit from it over time.

In this post I want to talk about what work was done, what the repercussions
are for our users, and what new stuff I've been working on to keep this ball
rolling.

IMCCompiler provides an `invoke` vtable which compiles a string of code and
returns an Eval PMC. This is all for backwards compatibility, but is
deprecated. PDD31 (which is still a draft, but a well-regarded one) specifies
that compiler objects should use methods instead. `.compile()` and
`.compile_file()`. IMCCompiler provides these two methods and now they both
return PackfileView PMCs instead. So, by upgrading to the proper compiler
interface for your PIR compiler, you should be automatically upgrading from
Eval to PackfileView. Basically, you want to change this:

    $P0 = compreg 'PIR'
    $P1 = $P0($S0)

...into this:

    $P0 = compreg 'PIR'
    $P1 = $P0.'compile'($S0)

In the first example `$P1` is an Eval. In the second example `$P1` is a
PackfileView.
