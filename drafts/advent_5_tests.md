---
layout: post
categories: [Parrot, Advent2011]
title: Advent 5 - Testing
---

Ever since I started working with Parrot I've noticed something interesting
about the community: They were very interested in unit testing. Parrot alone
as a suite with over ten thousand tests (and I still feel like there are some
portions of the VM that are heavily under-tested). When I first joined Parrot I
had never written a unit test nor did I really understand the value of testing.
I was a newbie fresh out of school where such practical aspects as that were
never covered. Despite my unfamiliarity with testing, I very quickly decided it
was a good idea and that more tests can definitely make software more awesome.

Writing tests for Parrot and Parrot-related projects is quite easy because we
have the infrastructure for it. The easiest way to write tests, in my opinion,
is with Rosella, but we have a Test::More library as well that can be used for
great effect and is the primary testing tool used in Parrot's own test suite.

Writing tests for Parrot is a great way to get involved in the project if you're
a new user, a great way to get familiarized with the capabilities of the VM, and
a very big benefit to the project in any case. First let's talk about Test::More
as it's used in the Parrot test suite and then I'm going to talk about Rosella's
test offerings.

Test::More is a very simple TAP producer library that implements a few standard
test functions like `plan()`, `is()`, `ok()`, and a few others. You use
Test::More like this:

    .sub main :main
        .include 'test_more.pir'
        plan(5);
        ok(1, "This test passes");
        ok(0, "This test fails");
        is(1, 1, "These things are equal");
        isnt(1, 0, "These things aren't equal, test passes");
        is(1, 0, "These things aren't equal, so the test fails");
    .end

The test harness reads the TAP output from the test file, checks the plan,
checks the results of each individual tests, and gives you a readout of the
overall pass/fail status of the test.

Test::More is very simple, and if you've used a TAP library before the basics
of it should be very easy to grasp. Plus, if you look through the Parrot test
suite (t/ directory and subdirectories) you'll see plenty of examples on usage.

Rosella offers several test-related libraries as part of it's collection:
Test (a unit testing library), Harness (a library for building test harnesses)
and MockObject (a mock-object testing extension) are all standard parts of the
Rosella lineup. There's also an experimental Assert library that adds some new
testing features as well.

Rosella ships with a default testing harness called `rosella_harness` which is
available when you install Rosella. You can run it at the commandline with a
list of directories like this:

    $> rosella_harness t/foo t/bar t/baz

The harness will run through all `*.t` files in the given directories, reading
the she-bang line in the file and using that program to execute the file. This
is the fastest way to get started with testing. Of course, you can also use the
Harness library to build your own harness in only a few lines of winxed code:

    $include "rosella/harness.winxed";
    function main[main]() {
        var harness = new Rosella.Harness();
        harness
            .add_test_dirs("Automatic", "t/foo", "t/bar", "t/baz", 1:[named("recurse")])
            .setup_test_run(1:[named("sort")]);
        harness.run();
        harness.show_results();
    }

The listing for a harness written in NQP is almost as short, and I've
[shown it several times][nqp_harness] on this blog before.

Writing a unit test file with Rosella Test is similarly easy:

    $include "rosella/test.winxed";
    class MyTest {
        function test_one() {
            self.assert.equal(1, 1);
    }
    function main[main]() {
        Rosella.Test.test(class MyTest);
    }

Each method in the MyTest class will be run as a test. Each test method may
contain zero or more assertions. If all assertions pass, the test passes. If any
assertion fails the whole test method immediately aborts and is marked as having
failed. Unlike Test::More above, we don't need to explicitly count the number
of tests for `plan()`. Instead, the library counts the number of methods for
us automatically.

Rosella's MockObject

