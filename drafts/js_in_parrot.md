---
layout: post
categories: [Parrot, JavaScript]
title: JSOP, JavaScript On Parrot
---

I was looking at the CorellaScript project the other day, and wanted to try to
tackle the same problem in a different way. This isn't an insult against
CorellaScript, but I know a little bit more today than I did at the beginning
of the summer and some of our tools have progressed along further than they
had at the point when CorellaScript was designed and started. I wanted to see
if we could convert to Winxed as an intermediary language, since Winxed is
syntactically similar to JavaScript already in some ways, and since Winxed
already handles most of the complicated parts PIR generation.

My idea, in a nutshell, is this: We create a JavaScript to Winxed compiler in
JavaScript, using [Jison][] and [Cafe][]. Jison is an LALR parser generator
written in JavaScript, and Cafe is an old project to use Jison to compile
JavaScript into JavaScript. At first, compiling JavaScript to itself doesn't
sound like such a great thing to do, but if we do some basic tree
transformations and make a few tweaks to the generated code suddenly it's
producing Winxed instead of producing JavaScript. Now all we need is an object
model and a runtime, and we have a basic stage 0 compiler for JavaScript on
Parrot.

Over the weekend, when we were trapped in doors because of the hurricane, I
put some of these ideas to the test. By the end of the weekend I had a new
project called JSOP (JavaScript-On-Parrot. It's a lousy name. I need a better
one). Today, the stage 0 JSOP compiler is parsing a decent amount of basic
JavaScript and has a small test suite. JavaScript doesn't have classes like
other languages do, so I had to add in some support to Rosella.Test to handle
javascript tests. Now that I've done that, we can use Rosella.Test to write
tests for JavaScript. Here's an example test file that I just committed:

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

That test file compiles down to the following Winxed code:

    function __init_js__[anon,load,init]()
    {
        load_bytecode('./stage0/runtime/jsobject.pbc');
    }

    function __main__[main,anon](var arguments)
    {
        try {
            load_bytecode('rosella/test.pbc');
            Rosella.Test.test_list(new JavaScript.JSObject(null, null, function (t) {
                    t.assert.equal(0, 0); }:[named('test_1')], function (t) {
                    t.assert.expect_fail(function () {
                    t.assert.equal(0, 1); }); }:[named('test_2')], function (t) {
                    t.assert.is_null(null); }:[named('test_3')]));
        } catch (__e__) {
            say(__e__.message);
            for (string bt in __e__.backtrace_strings())
                say(bt);
        }
    }

Formatting is kind of ugly right now, but it does the job. Executing this
file produces the TAP output we expect:

    <jsop git:master> ./js0.sh t/stage0/01-rosella_test.t
    1..3
    ok 1 - test_1
    ok 2 - test_2
    ok 3 - test_3
    # You passed all 3 tests

So that's not a bad start, right?

The stage 0 JSOP compiler is very simple, and I hope other people will want
to hack on it. I've borrowed, with permission, code from the Cafe project to
implement the parser. Cafe comes with a JavaScript grammar for Jison already
made, and some AST node logic. I added a new tree format called WAST which
is used to generate the winxed code. I modified the Cafe AST to produce WAST,
and deleted all the other code generators and logic from Cafe.

It all sounds more confusing than it is. The basic flow is like this:

    JavaScript Source -> Jison AST -> WAST -> Winxed

Winxed converts it to PIR, and execution continues from there.

So what still needs to be done? Well, lots! I've only implemented about 25% of
a stage 0 compiler, so most of the syntax in JavaScript is not supported yet.
I've only implemented enough to get a basic test script running (functions,
closures, "new", string and integer literals, etc). Basic control flow
constructs and almost all the operators are not implemented yet. I've also
implemented a basic runtime, but I don't have any of the built-in types like
Arrays yet, or most of the nuances of the prototype chain, etc.

The ultimate goal is a bootstrapped JavaScript compiler. Once we have stage 0
being able to parse most of the JavaScript language and execute it, we need
to create a stage 1 compiler written in JavaScript. It can borrow a lot of
code from Stage 0 (including the Jison parser). For that, we're going to need
PCRE bindings, among other runtime improvements. When we can use Stage 0 to
compile a pure JS stage 1 compiler, we self host and it's mission
accomplished. We've got a long way to go still, but I think this is a
promising start and I'm happy with the quick rate of progress so far. I'm
looking for people who are interested in helping, so please get in touch (or
just create a fork on github) if you want to help build this compiler.
