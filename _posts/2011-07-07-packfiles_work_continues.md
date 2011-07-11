---
layout: post
categories: [Parrot, Packfiles]
title: Packfiles Work, Continued
---

I merged in my big packfile branch yesterday, after approval from kid51 and
cotto. There were a few build failures manifesting on certain platforms, but
as of this morning those are fixed. The branch I merged adds in a new PMC type
called PackfileView. PackfileView is the intended replacement for the old Eval
type. It is a thin wrapper around the `PackFile*` structure and provides
several methods which are thin wrappers around packfile subsystem API
functions. With PackfileView, PIR users now have access to the same kinds of
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
that compiler objects should use methods instead: `.compile()` and
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
PackfileView. Eventually the `invoke` vtable on IMCCompiler will be deprecated
and removed, so it's good to get out in front of this train sooner than later.
This is a deprecation that we aren't intending to rush through, precisely
because there are a few parts to it and we want to make sure everything works
for all our users.

PackfileView is in master, along with all the deprecation notices for the
Eval PMC and the old packfile subsystem API functions.  This means we can
start updating those things relatively soon if we wanted, although like I said
we aren't in a huge rush. It's always good to know we have this kind of stuff
available for the supported release, however.

Before I merged the `whiteknight/packfilewrapper` branch, I started a new
branch from it called `whiteknight/pbc_pbc`. In this second branch I'm
continuing the work I started, doing the parts that really weren't mergable
in the week before a big supported release. I don't have well-defined goals
for `whiteknight/pbc_pbc`, I am going to do as much work as I possibly can in
it to clean up packfiles with two limitations: I don't want to run afoul of
any deprecation issues, since we can't do any of that until 3.9, and I want to
be able to merge before 3.7. Other than that, it's just a matter of how much
my little fingers can type.

In `whiteknight/pbc_pbc` I have two main targets: The first are the packfile
"pragmas", flags like `:init` and `:load` which are currently handled in a
very ugly, very brittle, and very complicated way. I asked a question on
the parrot chatroom yesterday about the purpose of one particular flag (The
flag was 'PBC_PBC', hence the name of the branch), and some of our brightest
and most experienced developers couldn't give me an answer. It turns out that
the flag seems to do nothing except loop over all constants in the packfile
and perform no action on them. Ripping them out is a performance boost, albeit
a small one.

Parrot has a really complicated way for dealing with various PIR behaviors
like `:init` and `:load`. IMCC, when it compiles Sub PMCs from PIR code, sets
flags on the PMC for init and load pragmas. Parrot, at various times in the
program load and execution cycle, will trigger these functions to execute.
What Parrot currently does is, when a packfile is loaded in or executed, is
to loop over all constants, find the Sub PMCs, test the Sub flags to see if
there are any `:init` or `:load` flags, and execute them if it's the right
time to do so. The function `do_sub_pragmas` takes flags to indicate what kind
of event we are doing, and performs this loop. For instance, calling
`do_sub_pragmas` with the `PBC_LOADED` flag will execute all Subs with the
`SUB_FLAG_PF_LOAD` flag (`:load`) set. Why we need two flags for this purpose,
and we can't just use a single flag is beyond me. The `PBC_MAIN` flag triggers
`SUB_FLAG_PF_INIT` (`:init`) Subs. It used to find and cache the `:main` Sub
too, but because of other recent improvements to packfiles it doesn't do that
anymore. The flags `PBC_IMMEDIATE` and `PBC_POSTCOMP` handle the `:immediate`
and `:postcomp` Subs, respectively, and are only called from inside IMCC. Why
IMCC needs to use flags and packfile-API routines to trigger functions it has
just compiled is another mystery, and is a huge source of broken encapsulation
for the system. The flag `PBC_PBC`, which I mentioned earlier, used to cache
the location of `:main` the same as `PBC_MAIN` did, but wouldn't trigger the
`:init` functions. Now that the `:main` Sub handling is improved, `PBC_PBC`
appears to be unnecessary and has been removed in my branch.

Confused yet? If you aren't scratching your head yet, you will be.

Parrot calls `do_sub_pragmas` all the time with different flags for different
things: When a PIR program is compiled and executed (PBC_IMMEDIATE,
PBC_POSTCOMP, PBC_MAIN), when a PIR code snippet is compiled with the PIR
compreg (PBC_IMMEDIATE, PBC_POSTCOMP), when a PIR code snippet is compiled and
loaded with the `load_bytecode` op (PBC_IMMEDIATE, PBC_POSTCOMP, PBC_PBC),
When a .pbc file is loaded with the `load_bytecode` op (PBC_LOADED), when
a PIR file is compiled using the old embedding API functions (PBC_IMMEDIATE,
PBC_POSTCOMP, maybe PBC_PBC too), when a .pbc file is loaded using the old
embedding API functions (PBC_LOADED, maybe PBC_PBC too), etc. On top of that,
the system previously used to only support one packfile: Every time you
compiled a file or loaded a library the packfiles were merged. So, what Parrot
does is to *clear the flags* on the PMC *constants* in `do_sub_pragmas` to
prevent subs with multiple flags, or subs which might have been triggered at
multiple times, from being triggered more than once.

Here's the train of thought:

1. Parrot should automatically and apparently magically trigger certain Sub
   constants at certain times in response to certain actions. These Subs
   should be executed in no apparent or user-controllable order, should not be
   allowed to take arguments or return values, should execute in a separate
   internal runloop making error reporting and other issues difficult, and
   this behavior should not be able to be disabled, modified, postponed, or
   tweaked in any way by the users.
2. Because Parrot doesn't know when the User might want to be triggering these
   functions, and because it's very possible that we could magically try to
   trigger them twice or more (since it's all out of the control of the user),
   we need to clear the flags in a *constant* PMC in the packfile. This means
   that doing something trivial like compiling a PIR file and trying to write
   it out to .pbc might not contain all the semantics you intend, or we might
   require other ugly hacks to preserve them.
3. ??? (something we don't understand)
4. Profit!

Sound crazy yet? Luckily, I'm planning to change all that. Instead of having
`:load`, `:init`, `:main`, `:immediate` and `:postcomp` flags for all sorts
of slightly different situations where Parrot should magically and
automatically execute a function beyond user control, I'm planning to have
arbitrary string tags for functions:

    .sub 'foo' :tag("load")
        ...
    .end

That's a long term goal, but anybody can see that there is a lot more
flexibility to that approach than needing to hardcode in a new flag definition
to the IMCC system every time somebody is unhappy with the current crappy
flag options. In this kind of system, Users can name their flags whatever they
want. This is complimented by a new feature of the PackfileView PMC to get a
list of all subs from the packfile with a given tag:

      $P0 = new ['PackfileView']
      $P0.'read_from_file'('foo.pbc')
      $P1 = $P0.'subs_by_flag'('init')
      $P2 = iter $P1
    init_top:
      unless $P0 goto init_end
      $P3 = shift $P2
      $P3()
      goto init_top
    init_end:

The new `:tag()` flag has not yet been added to IMCC, but it could be added
relatively soon if I can muster up the courage to modify IMCC internals like
that. The `.subs_by_flag` method has been added to PackfileView in my new
branch, however.

Another necessary component is to get Parrot to stop automatically executing
subs for you when it thinks you want it. I mentioned the `load_bytecode` op
before. `load_bycode` in master takes a single string argument of a bytecode
file to load, and searches through all the library load search paths to find
and load it. Actually, if the file has a .pir extension it searches for the
file and then compiles it, triggering PBC_MAIN. If it has a .pbc extension,
it just loads it triggering PBC_LOADED. This discrepancy forces most Subs
generated by tools like PCT or NQP to be tagged with `:load :init` to make
sure the damn Sub executes no matter whether it was loaded as a .pir or a .pbc
file. In my branch, I've added a new variant of `load_bytecode`, which returns
a PackfileView:

    load_bytecode "foo.pbc"         # OLD version (BAD)
    $P0 = load_bytecode "foo.pbc"   # NEW version (GOOD)

The first opcode with the one argument is going to be deprecated and removed
eventually. I will probably put the notice in before the 3.9 release. The
new variant performs *ZERO* magical behavior: It searches through the search
paths and loads the .pbc file. It does not automatically compile a .pir file.
It does not automatically trigger `:load` or `:init` functions. It returns you
a PackfileView on success, and you have the tools necessary to *perform those
actions yourself*. If you want to trigger the `:init` functions, you can do
that. If not, don't. Same with `:load`.

A big big big advantage to this situation is that if you use this method, all
your `:load` and `:init` functions will execute in the master runloop, instead
of recursing into a new runloop. This creates huge performance savings, and
is a big win for stability. Not to mention the fact that you can get more
creative with these functions: Take parameters, return results, use a
coroutine to prevent initialization behavior from happening more than once,
etc.

As has become customary in these kinds of posts, here is an example of what a
simplified Parrot front-end written in PIR would look like:

    .sub __main :main :anon
        .param pmc args
        .local string prog_name
        prog_name = '__parse_args'(args)

        .local pmc pir_compiler
        .local pmc packfileview
        pir_compiler = compreg 'PIR'
        packfileview = pir_compiler.'compile_file'(prog_name)

        push_eh _handler

        .local pmc init_subs
        init_subs = packfileview.'subs_by_flag'('init')
        '__trigger_init_subs'(init_subs)

        .local pmc main_sub
        main_sub = packfileview.'main_sub'()
        push_eh _handler
        main_sub(args)
        pop_eh
        exit 0
      _handler:
        .local pmc ex
        .local int exit_code
        pop_eh
        exit_code = '__handle_exception'(ex)
        exit exit_code
    .end

I'm obviously leaving out several options, like loading in or writing out
.pbc files. I'm also hiding some implementation details inside subroutines.
For running PIR programs this example program should be functionally complete.
We are extremely close to being able to write the Parrot frontend in PIR, and
I might be able to start putting such a thing together in a matter of days or
weeks, not months as I had originally planned.
