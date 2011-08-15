---
layout: post
categories: [Parrot, Winxed]
title: Multiple Dispatch in Winxed
---

Here's a snippet of Winxed code I was running a few days ago on my machine:

    function Foo(int a, int b) { ... }
    function Foo(string s) { ... }

    function main[main]() {
        Foo(1, 2);
        Foo("Hello");
    }

Notice anything interesting about it? Up until now, Winxed hasn't supported
multiple dispatch. NotFound didn't really use the feature and users haven't
been asking for it too loudly, so it never got implemented. However, now that
I'm pretending to be a Winxed super haxxors, I decided to take a stab at it.
As of yesterday evening, I have a working prototype and am doing some testing
and tweaking to get a pull request ready.

This isn't full-featured MMD, yet. Parrot's MMD system allows you to dispatch
based on types and inheritance and there are wildcards. The current Winxed
implementation I'm playing with only dispatches on the four primary register
types. We read the parameter list and, if there are multiple functions with
the same name in the same scope, we convert them to multis.

It's a simple patch and just the first step towards getting full Multi
support. The hardest part about moving forward is not the implementation
(I'll reiterate, Winxed is pretty easy to hack on), but instead picking the
syntax we want to use to specify options.

NotFound has been away, and I don't think that this will get merged in to
master or pulled into the Parrot repo before the release tomorrow. If it
passes code-review muster, maybe it could be in place shortly therafter. Then,
we can start on the next step: Finding a syntax with which we can specify
improved type information supported by the MMD system.

As a quick exploration, we could do something like this:

    function Foo(Bar.Baz baz, Bar.Fie fie) { ... }

That would be just fine for a multiple dispatch situation, but specifying
types in the signature implies that we are doing some kind of type-checking,
which I suspect Winxed would not want to do. If we did that, we could
automatically promote every single function with type information in the
signature to a MultiSub, even if there was only one of them. Right now, the
patch only  auto-promotes a Sub to a MultiSub if there are more than one of
them with the same name. This promotion, and the dispatch mechanisms
associated with MultiSub have non-negligible cost. It runs contrary to
expectations to think that adding more type information for the compiler would
*decrease* performance of the generated code.

If we keep the logic that we only promote to MultiSub when there are multiple
functions with the same name, we could instead insert type checks into an
ordinary Sub that has type information but is not auto-promoted. Of course,
then the code generator for Sub has to keep track of storage information in
the owner namespace, which gets messy quick. The laziest approach would be to
not insert any type-check information at all, and allow a parameter which is
declared as `Bar.Baz baz` to be filled by an object of any type without any
indication that it does not match what is written. I suspect that's not what
anybody wants.

Perhaps we could do something like adding metadata in tags:

    function Foo[multi(Bar.Baz, Bar.Fie)](var baz, var fie) { ... }

But that's verbose and ugly, and updating the parser to support it is
non-trivial. It does have the benefit that you are explicitly telling the
compiler that you want it promoted to MultiSub.

Another possible syntax would be this:

    multi Foo(Bar.Baz baz, Bar.Fie fie) { ... }

Here, we use the `multi` keyword if we want to specify type information in
the parameter list, to make clear that we only do type checking in a MultiSub,
but don't do it for an ordinary `function`.

One more than just came to mind would be something like:

    function Foo(var[Bar.Baz] baz, var[Bar.Fie] fie) { ... }

Here, it's clear that the first parameter is still a `var` and can be any
type, but there is the tag there that says it should be considered a specific
type in a multidispatch situation.

Anyway, the easy part is done. With my patch you can do basic multi-dispatch
on the four register types in Winxed without any new fancy syntax. That's
easy, it's implemented, and it works. Doing the next step is harder because
we are going to need to add new syntax, and finding such a syntax which is
functional, attractive, and does not promise things it does not deliver.

Maybe we don't find any such syntax, and Winxed never gets an easy syntax for
class-based multiple dispatch. That's a little disappointing to think about,
but we've come pretty far without any MMD support in Winxed, and we will go
much further with the little bit provided in my patch. Maybe that takes us far
enough for most uses.

