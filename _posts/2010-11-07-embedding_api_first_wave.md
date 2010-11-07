---
layout: post
title: Embedding API, First Wave
categories: [Parrot, Embedding]
---

I skimmed through the Parrot executable code a few days ago and gathered up
a list of the libparrot functions called from it. This is going to help us
break the
[embedding API](/2010/11/06/embedding_api.html)
work into a series of waves that we can tackle one at a time. I like working
on big tasks in small chunks anyway, but Parrot's monthly release schedule
basically mandates that we have certain windows in which we can deliver
incremental improvements. It's my number one priority to ensure that Parrot's
users get these API improvements as quickly as we can possibly manage.

The first wave, which got this entire train rolling, was inspired by
[shockwave's comments](/2010/10/21/product_management_team.html#3747058508749526829)
to a prior post where he talked about his difficulties with the embeddng API.
I already have a function in place to resolve the one problem he mentioned
specifically. That function, `Parrot_load_bytecode_file` needs to be renamed
and moved to a new file of official embedding API functions. This will
certainly be in place by the 2.10 release on November 16^th^. The big result
that comes from this first wave is the creation of the
[Product Management team](http://trac.parrot.org/parrot/wiki/ProductManagementTeam)
and the [Embedding API task](/2010/11/05/embedding_api_team.html) force. We're
also using this opportunity to do some proper, forward-thinking design, so th

In my mind the second wave will involve fixing the Parrot executable so that
it only interacts with libparrot through actual API function calls. This is
going to be a little bit more tricky, but I have full confidence that we can
get it done by the 2.11 release on December 21^st^.

I have prepared a list of all libparrot functions that are called from the
Parrot executable, and a list of all structure fields which it pokes into
directly. All of these will have to be converted to new API calls (some of
which may be nearly identical to the previous API functions).

First, two macros are called:

    PARROT_BINDTEXTDOMAIN
    PARROT_TEXTDOMAIN

I'm only vaguely familiar with the purposes of these macros, though I don't
think they are really used by Parrot as it currently exists. I'm not sure that
we need them right now (though it probably doesn't hurt). I think we should
probably turn these into functions instead of macros. The NCI module from
other interpreters won't be able to bind to a macro, for instance.

Next, the functions that are called:

    Parrot_set_config_hash
    allocate_interpreter
    initialize_interpreter
    Parrot_set_trace
    Parrot_set_runcore
    Parrot_set_executable_name
    imcc_run
    imcc_run_pbc
    Parrot_destroy
    Parrot_exit
    longopt_get
    Parrot_io_printf
    Parrot_warn
    Parrot_lib_add_path_from_cstring
    Parrot_set_warnings

All these functions will, at the very least, be renamed to `Parrot_api_...`
and in many cases will change substantially to include some of the mechanisms
and design points that I talked about in
[yesterday's post](/2010/11/06/embedding_api.html).

Finally, the structures that are directly touched:

    interp->gc_sys->sys_type
    interp->gc_threshold
    interp->hash_seed
    interp->output_file

The deprecation policy requires that all pointers and structures be
considered opaque, so we absolutely can not be exposing them like this.
It doesn't matter *what* the interface is, so much as it matters that we have
*any* interface. The current situation is unmaintanable and unacceptable.

The third wave is a little bit more nebulous, but at the same time it is
probably the most important: we need to allow embedding applications to work
directly with PMC and STRING types, and expose all the necessary operations
on those types. I don't know what the timeframe is going to be for this third
wave, we need to gather a lot more information, feedback, and user
requirements before we can even start designing this portion of the
interface.
