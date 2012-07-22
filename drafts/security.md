---
layout: post
categories: [Parrot, Security]
title: A Security System
---

What does a security system look like? It's really worth taking some time to
ask this question first, before we start to implement one. In this post I'll
describe my take on such a system, and hopefully use that as a mechanism to
start a larger conversation and gather some important feedback.

Let's start by asking what a security system *does*. A security system should
represent some kind of storage mechanism for holding information about
ownership, permissions and restrictions. In otherwords, at it's heart it's a
simple metadata storage system. During certain operations I should be able to
ask the permissions system "Am I allowed to do this?" and receive back a boolean
true or false. If true, I proceed to do it. If false, I don't (and maybe throw
an exception). The "this" I'm referring to could be any number of things from a
known list of built-in action flags, or custom named permissions at the user
level. The key to making such a system work is to provide an interface for the
various security-related operations, and to use that interface consistently
inside Parrot core and our important extensions.

Another important consideration is that the permissions system should be
readable (and quickly so, to avoid a performance penalty), but it should not
be writable in most cases. You don't want a system where a user can ask "do I
have this permission?" and, if not, simply add that permission for themself.

It's worth pointing out, at this point, that security in Parrot is a tricky
beast for a lot of reasons. Let's say that I have a restriction for performing
socket IO. If the flag is on the user can create and use sockets like normal.
If off, the user cannot create or use sockets. This seems all well and good
until a malicious user creates and loads a custom dynpmc which performs socket
operations without checking or enforcing permissions. In that case, the user
gains access to networking operations despite the security permissions set. To
avoid that kind of problem, we'd also need to have and enforce permissions
against loading dynpmcs, dynops, and even NCI. Once you have access to NCI,
for instance, it's a small matter of loading in socket-related functions and
calling them directly, which obviously would bypass many security-related
restrictions. In other words, shutting off access to Sockets (for example)
necessitates shutting off a variety of other features as well, especially those
which dynamically load binary-level libraries and extensions.

In even the most basic of security-related systems, we would therefore have to
have at least two levels: Unlocked (no restrictions) or minimally-secure
(no dynamic loading, no NCI). Once the security context is set to at least a
minimum level of security, the only binary libraries you'd be able to use are
the ones already available. The only NCI routines able to be created and
called would be the ones already prepared. Once you close the lock, those
things can't be done anymore. You do run into yet another loophole that if
a hapless administrator turned off things like socket IO, but had already
provided a reference to socket-related NCI routines, you'd be able to use
those things despite the restrictions, because there's no way for the system
to know that a particular NCI routine is "socket-related" or not. At least, not
without the user supplying that metadata. And if the user knows to use that,
they may know enough not to make those things available in the first place.

Even if you take the naive approach that you can hard-code in a list of
functions that are no-go, that doesn't solve anything. A malicious user could
still write and build a custom library which performs socket-related
networking operations, but with function names which don't match what's on the
function name blacklist and then you're no better off. Again, if you want to
impose any security restrictions at all, from the most mundane to the most
critical, you need to shut down huge swaths of the VM.

In any security system we create, therefore, we're going to need at a minimum
a way to turn off dynamic library loading AND some clear guidelines to inform
administrator best practices. No matter how great the system is that we end up
with, so long as we have NCI AND uninformed administrators cargo-culting code
snippets without fully understanding the ramifications, there's going to be
some trouble.

Moving on...

We'll start off with a `SECCONTEXT` structure, which represents a security
context. These will be related, conceptually, to an ordinary call context,
although it will probably operate more at the interp level than anything. In a
basic setup, we'll have a security context pointer on the interp which will
point to the current context. When we create a child interp we'll have the
option of assigning a new context to that interp with a more restricted
`SECCONTEXT`. Notice that any given context may only create a context that is
*more restricted* than itself. If my context doesn't have file permissions,
I cannot create a context that does either. The top-level parent interp would
have all permissions, and an API to create a new context and to remove
permissions from a context. There is no API to add a new permission to a
context that doesn't already have it.

When you create a new `SECCONTEXT` gets a copy of the permissions from its
parent. Then you can remove permissions from it if necessary. You can never
add new permissions to a pre-existing `SECCONTEXT`.

The `SECCONTEXT` structure will contain a few things: A pointer to a parent
context for pushing/popping, a field or two devoted to bit flags, and possibly
a hash for named permission entries and some other things. It might also
contain a pointer to the owner interp, in case we pass these security contexts
around we want to make sure the current interp is checking permissions against
a security context that it owns.

In general operation we have a single function worth remembering:
`Parrot_sec_has_permission` or maybe something exception-based like
`Parrot_sec_assert_permission`. Regardless of whether or not the function
returns a boolean or throws an exception on failure the result is clear:
assuming that these things are used consistently, you won't be able to
perform an action that you don't have permissions to perform.

In this basic formulation each interp has a security context, and child
interps would have contexts with the same permissions or fewer only. The only
way to create a new top-level interp (and therefore a new fully-unlocked
security context) would be from the embedding API in C. We'll go ahead and
make the assumption that if the malicious user has access to the C compiler
and a terminal from which to compile it, the system is already compromised
and Parrot would do well to just play along like nothing is wrong.

By default, I think a really good rule of thumb is that only the top-level
interp should have the ability to load dynpmc, dynops and NCI libs. Child
interpreters, by default, should probably have these actions restricted.
Whether that's a rule we enforce in code or whether we rely on convention is
something to be decided later.

So let's say we have a function `Parrot_sec_assert_permission`, and we wanted
to implement an IO routine to read data from files. Maybe we do something like
this:

    Parrot_sec_assert_permission(interp, PERMISSION_FILE_IO, "file read");

If the permission exists, there's no harm and no foul. We continue to read
like normal. If the permission does not exist instead we might see an
exception like this:

    Security Error: Do not have permission 0x01 "file read"

Of course, now that I look at it, we probably do need that function for
testing a permission and returning a boolean as well. In either case, that's
at most two functions you would need to keep track of to use the security
system effectively.

Creating a new interp with a restricted security context would be able to be
done at any level, but would probably happen most frequently at the bytecode
level. Here's an example in winxed of what such an interface could look like:

    var new_interp = get_interp().create_new_child();
    new_interp.remove_permission(0x01);
    ...

And after you remove the permission any bytecode executed by that interp would
be unable to do file IO.

Off the top of my head, here are a few things that I think we would want to
be able to control as part of any basic security subsystem:

1) As mentioned above, the ability to turn off loading dynpmcs, dynoplibs,
   and NCI.
2) The ability to load bytecode at all, and a possible whitelist/blacklist of
   paths from which bytecode can be loaded if loading is possible.
3) The ability to load HLL packages, which are like loading bytecode except
   they may also contain dynpmcs and dynoplibs as part of a package (and, as
   above, a whitelist/blacklist from which they may be loaded).
4) As also mentioned above, a flag to prevent file IO (or, two flags for read
   and write). Also, a whitelist/blacklist of paths in which IO is possible.
5) Likewise, flags for Socket IO and a possible whitelist/blacklist of either
   domains, IP addresses and/or IP subnets.
6) An ability to set a recursion limit. We already have this at the interp
   level, but moving it into the security system and making it optional seems
   like a much better idea, since in many cases we don't want a recursion limit
   imposed unnecessarily.
7) The ability to create child interps and, therefore, to set permissions on
   them.
8) Some sort of limit on the amount of memory able to be allocated, or the
   number of PMC/STRING headers able to be allocated, and/or the maximum
   allowable size of an individual buffer allocation.

I've also been kicking around the idea of having a hash of named values able
to be set at the user-level. The user could create a custom named permission
with some associated data. The permission would only be able to be set and
modified from the parent interp, but once you're inside the child interp they
are read-only. An HLL or library developer would be able to do something like:

    get_interp().assert_permission("can_do_stuff");

    var secdata = get_interp().get_permission_data("can_do_stuff");
    if (secdata == ...) { }

So those are just my initial thoughts about a security system for Parrot. I'd
really like to hear some feedback from other people who have particular
security-related needs or wants. I have a few other things on my TODO list
before I can even think about starting work on a new security system, so
we have plenty of time to put a plan together and get things right.

With this system, the implementation of the data structurs and the API will be
a relatively small issue. Using that API consistently in all the places it's
necessary, and doing comprehensive testing to make sure we lock things down
correctly in all cases is much much harder. We'll cross that bridge when we get
to it.
