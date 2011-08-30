Yesterday I introduced JSOP, a new attempt at a JavaScript-on-Parrot compiler.
JSOP is still very early in its development, but it's able to translate enough
JavaScript code to Winxed to host a test suite using Rosella.Test. The suite
is still small, but it will be growing as new features are added.

There are three general paths of development ahead of us to get a working
stage 0 compiler capable of bootstrapping a stage 1: Improve the compiler,
write an object model, and write a sufficient runtime library.

The compiler work sounds complicated, but it's very straight-forward. For
every AST node coming out of the Jison parser, I transform it into a sequence
of new WAST nodes, and from there I tell each node how to generate proper
Winxed code. This process is far simplified because I don't need to deal with
things like type checking, register allocation and other details. My stage 0
compiler doesn't even need to maintain a symbol table or at least not more
than a basic one. I let Winxed tell me if the code I generated has everything
defined that needs to be. That work is straight-forward and easy for other
people to get involved with.

The second path is a little bit more complicated, because much of the core
of the JavaScript language is in the semantics of its object model. Getting
the interaction between objects and prototypes working correctly, or at least
approximating it closely enough for bootstrapping purposes, is extremely
important. I've started work on the object model, although it's slow going and
I'm going to need a lot of tests. Before I even get that far, I need to learn
a heck of a lot more about the internals of what a JS object model is supposed
to do.

At the moment in JSOP stage 0, all objects are going to be wrapped up in
instances of the `JSObject` class. What differentiates each JSObject is the
constructor used to create it, and the prototype that it draws default
behavior from. The stage 0 object model is not going to be the pinnacle of
efficiency or completeness, but it should be enough to get the job done.
Hopefully, stage 1 runtime and later will be able to be built upon 6model, and
get all sorts of improvements from that.

For stage 0 the object model does not need to be perfect or complete. It
simply needs to support enough features and correct semantics to bootstrap to
stage 1. The Stage 1 compiler can easily be written in a subset of JavaScript
which is supported by Stage 0. Finding such a subset which makes
implementing Stage 1 easy without making Stage 0 too complicated is going to
be a very interesting search.

The runtime we are going to need consists of things like global functions,
global objects, common methods, and storage semantics, etc. One big thing we
are going to need if we ever hope to bootstrap is PCRE bindings for the Jison
parser and regular expressions. That's hopefully not too much work, assuming
the PCRE bindings that ship with Parrot haven't gone completely bitrotten.
We're going to need storage semantics as well. For instance, JavaScript has
a notion of global variables that we need to cope with. The naive solution
would seem to be that we disallow global objects and require all objects to
have function scope, like what Winxed requires. However, that fails for even
simple cases:

    var array = new Array();
    var array_prototype = Array.prototype;

Here, Array is a global object which is both invokable and has a prototype and
other features which can be examined. It really needs to be made available as
a global variable unless we want every reference to Array to be treated as one
big hack. So that means all top-level functions and variables not defined with
the `var` keyword are going to need to be stored in some global cache.







