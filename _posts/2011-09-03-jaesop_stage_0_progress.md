---
layout: post
categories: [Parrot, Jaesop]
title: Jaesop Stage 0 Progress
---

A few days ago I started the Jaesop project (formerly "JSOP") to explore
creating a JavaScript compiler on Parrot using bootstrapping. After only a few
days of real effort I'm getting pretty darn close to having a stage 0 compiler
ready for use.

The Jaesop stage 0 compiler, called `js2wxst0.js` translates JavaScript code
to Winxed. It is not a full JavaScript compiler; instead it's a useful subset
of JavaScript which can be used for bootstrapping. Most of the syntax is
supported, and the object model has acceptably faithful semantics. What I
don't have is complete support for all built-in object types and methods, or
100% complete syntax translation. Some things like the `with` keyword are not
and will not be supported, for example. The compiler doesn't currently handle
some common bits of syntax like `try`/`catch`, `switch`/`case`, or a few other
things. Many of the basics like operators, assignment, variables, functions,
closures, and basic control flow (for, while, if/else) are working just fine.

Of course, if it did everything and was perfect, we wouldn't call it "stage 0"
we would just call it "the JavaScript on Parrot Compiler". The stage 0
compiler isn't the end goal, it's just a tool we're going to be able to use
to make a better compiler. I'm not looking to make something perfect here,
I'm trying to put together a bootrapping stage 0 as quickly as possible.

The stage 0 compiler architecture is very simple. The Jison parser outputs
AST, which I've had to make only a handful of modifications to from the
original Cafe source. Then, the AST is transformed into WAST, a syntax tree
for creating Winxed code. Finally, the WAST outputs Winxed. Most of the code
here is complete and working very well. Late this week I finished the basics
of the object model, and then I updated the compiler to output correct code
for the model, and just today I got the test suite working again with all the
new semantics.

The test suite is up and running again, although it doesn't have nearly enough
tests in it to cover the work I've done until this point. The suite has tests
written in Winxed and also tests written directly in JavaScript. The former is
for testing things like the runtime, the later is for testing parsing and
proper semantics. I want to increase coverage in both portions, because I've
been dealing with a lot of aggravating regressions here and there as I code,
and I want to make sure things get better monotonically from here.

Getting the test suite to work with the real JS object model was a little bit
tricky. To get an example of why, here is a test I had in the suite prior to
todays hacking:

    load_bytecode("rosella/test.pbc");
    Rosella.Test.test_list({
        test_1 : function(t) {
            t.assert.equal(0, 0);
        },
        test_2 : function(t) {
            t.assert.expect_fail(function() {
                t.assert.equal(0, 1);
            });
        },
        test_3 : function(t) {
            t.assert.is_null(null);
        }
    })

Basically, this was my first sanity test, to prove that I could call Rosella
Test functions from JS code. Unfortunately, after I re-did the object model,
this test was broken. It got broken because of a fundamental feature of
JavaScript: methods are just attributes, except they can be invoked. So a call
to this:

    Rosella.Test.test_list(...)

After compiling to Winxed, looks like this:

    var(Rosella.*"Test".*"test_list")(...)

The `.*` operator looks up an attribute by name. The `var(...)` cast pulls
out the value of the attribute into a PMC register, and the parens at the end
invoke it. Notice that `Rosella.Test` isn't an object, it's a namespace. So
that code was broken. Also notice that JavaScript has a notion of a global
scope. We haven't explicitly declared a variable named "Rosella", so Jaesop
tries to do a global lookup:

    var Rosella = __fetch_global("Rosella");
    var(Rosella.*"Test".*"test_list")(...);

Also, inside the tests, the assertions are done with `t.assert.equal()`, etc.
But that's clearly wrong too, for all the same reasons. In short, the code
was broken.

After some fixing and refactoring, I have the test situation all sorted out.
Here is the same test today:

    var t = new TestObject();
    test_list([
        function() {
            t.equal(0, 0);
        },
        function() {
            t.expect_fail(function() {
                t.equal(0, 1);
            });
        },
        function() {
            t.is_null(null);
        }
    ]);

The methods `TestObject` and `test_list` are defined in the test library as
globals. `TestObject` is basically a JavaScript-ish wrapper around
`Rosella.Test.Asserter` with all the same methods.

The test suite is working but does need to be expanded. I have a few more
things to add to the compiler and runtime as well, which will be easier to do
with better test coverage. I very much intend to have a working and usable
Jaesop Stage 0 to release soon. Certainly it should be available by the Parrot
3.9 release, hopefully much earlier. With that available, I want to get
started on Stage 1. Stage 1 doesn't have to happen at the same blistering
pace. In fact, I think it would be beneficial for us to wait until Parrot has
6model support built in, so we can start making the "real" object model using
those tools instead.

