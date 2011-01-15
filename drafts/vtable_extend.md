---
layout: post
categories: [Parrot, Embedding]
Title: Problems with extend_vtable.c
---

A few days ago on IRC we were having a discussion about the new API, and the
role that PMC VTABLEs are gong to play in it. The old embedding API provided
a series of direct accessors to PMC vtables, in a file called
`src/extend_vtable.c`. Don't go looking for that file in the repo on github,
however. `src/extend_vtable.c` is automatically generated during the Parrot
build and contains a long series of simple wrappers for all VTABLE interface
functions.

I think this is a big problem, and the new embedding API doesn't provide any
such director accessors. At least, it doesn't provide such a large set of
them, and doesn't use a tool to blindly generate them. In this post, I'm going
to explain why.

VTABLEs are the very definition of low-level accessors. They are the
absolutely lowest level at which Parrot or any application running on top of
Parrot can reliably operate on arbitrary PMC types. For performance reasons,
sometimes Parrot avoids the VTABLE interface and pokes directly into the
external datastructures of a PMC, but it does this only in a small handful of
situations where Parrot knows precisely what type of PMC it expects. This
isn't a general mechanism, and in many cases attempting to use a Subclass of a
core type in place of the core type would cause serious breakage.

Some VTABLE interfaces provide very important operations. They provide
operations like push and pop for array types, and integer-keyed lookups. For
hash types they provide string- and pmc-keyed lookups. They provide the
ability to set and retrieve primitive values into a PMC, like setting the
INTVAL value of an Integer PMC, or reading out the `STRING*` value of a String
PMC. These are basic operations, and no API for Parrot is going to be
considered too functional without these things.

What the API wants to provide are *operations*, not necessarily any particular
interface for performing them. The interface only serves as an abstracting
mechanism where access to the operations is provided without exposing the user
to the internal design architecture of Parrot, or subjecting them to the
frequent changes in that architecture. The API can provide a push and pop
operation without needing to call that operation a "VTABLE" and without having
to explain to the poor embedder what a VTABLE is, and how it provides
interoperation.

Some VTABLEs are internal-only, and can be down right dangerous for an
embedder to attempt to use directly. A common and immediate example that comes
to mind is `VTABLE_invoke`. Here's a friendly reminder: *`VTABLE_invoke` is
not for you, it doesn't do what you want as an embedder. Don't call it*. The
same goes for VTABLE_mark, VTABLE_destroy, VTABLE_init, VTABLE_init_int,
VTABLE_init_pmc, and maybe a handful of others as well. We do need to provide
a mechanism for invoking an invokable PMC, but `VTABLE_invoke` is the
low-level primitive, not the tool for the end-user to use. VTABLE_invoke is
called by the helper routine, which is called by another routine, which is
called by the API function itself.

`VTABLE_freeze` and `VTABLE_thaw` are other good examples. They don't directly
do what you want either, they are just low-level primitives which are called
by the functions that do what you want: `Parrot_freeze` and `Parrot_thaw`. And
even in those cases, the names "freeze" and "thaw" don't necessarily make much
sense to you if you're new to Parrot. That's why the API calls these
operations "serialize" and "deserialize" instead, and adds a little bit of
error-checking to boot.

The API isn't for Parrot's core developers who know the system architecture,
know the terminology, and know the problems and pitfalls associated with
various routines. The API says "Here are the operations that Parrot can
perform", while leaving out all the details about *how* those operations are
performed. Those details aren't the business of the embedder, and shouldn't
be. Encapsulation means never having to share those dirty little secrets with
your users.

VTABLEs are also not exactly stable. They're better than many parts of the
system because they are so pervasive, but they still do change from time to
time. Sometimes VTABLEs are added. Recently, it's more common for them to be
culled. Occasionally they change in format or function. The operations that
Parrot provides do not change. Those are always implemented in another way
eventually. The only thing that changes are the specific primitive operations
that Parrot uses to implement those operations. I say we just go ahead and
shield the embedder from those kinds of changes and details.

`src/extend_vtable.c` doesn't have to go away. I think it might be a good
thing if it did go, but it isn't necessary. Embedders are obviously not the
only group of users of Parrot, there are also the extenders to think about as
well. Anybody who is writing custom PMC types in C, or custom Oplibs in C is
an extender, not an embedder. These people *are* going to need to know about
VTABLEs, and are going to need to be able to call them. Many extenders use the
`VTABLE_` macros directly which is fine for now but which may not ultimately
be what we want. Eventually we probably will want the VTABLE macros hidden
behind thin wrapper functions like those in `src/extend_vtable.c`, but it
isn't important right now. Even extenders still don't need to be calling
`VTABLE_invoke` or `VTABLE_share_ro` directly, so I'm on the fence about
whether we should be auto-generating our VTABLE wrappers or whether we should
bite the bullet and write them up by hand. I suspect we will be
auto-generating them always.

It's my opinion that the embedding API, probably moreso than any other
interface into Parrot or libparrot, needs to be stable. PIR has historically
not been very stable, but it's my sincere hope that people stop using PIR and
eventually start creating their own packfiles directly. In such a world, so
long as the API for creating packfiles is sufficiently abstracted and stable,
HLL users will get the same stability benefits as the embedders have with the
embedding API. Extenders are always going to be on the losing end of the
stability battle, at least until we take the time to design, implement, and
enforce a usable extending API. That day may never come, since the
capabilities of libparrot which extenders need to access is such a large and
varied list. It may be impossible for us to ever boil down a separate
extending API from the various internal subsystem APIs.

It's not the embedder's job to keep track of things like VTABLEs, or to know
the difference between a VTABLE and a method, or the difference between a
VTABLE and a PIR op. That's all minutia, and is not important. What is
important is for embedders to know that PMCs are data items, and can be
interacted with just like other objects or items from other programming
systems: You can perform operations on them, you can call methods on them,
you can create them, store them, fetch them, serialize and deserialize them,
and destroy them. They can hold data. They can perform various actions. They
can do a bunch of things if you pass them to to the right APIs. In the end,
what I want to provide is that set of "right APIs", and give our embedders the
tools they need without making them understand our internal architecture
first.
