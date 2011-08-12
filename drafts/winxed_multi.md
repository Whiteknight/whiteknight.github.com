Here's a snippet of Winxed code I was running earlier today:

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
