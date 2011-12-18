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

Several months ago we had Tapir, a simple TAP harness project written by
dukeleto and others in NQP. There was also a project called Kakapo written by
Austin Hastings which included several unit test and mock object utilities, also
written in NQP. I absorbed a lot of ideas from both projects (and eventually
rewrote those ideas in Winxed) for the Rosella Test, Harness and MockObject
libraries.

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

[nqp_harness]: http://whiteknight.github.com/Rosella/libraries/harness.html

Writing a unit test file with Rosella Test is similarly easy:

    $include "rosella/test.winxed";
    class MyTest {
        function test_one() {
            self.assert.equal(1, 1);
        }
    }
    function main[main]() { Rosella.Test.test(class MyTest); }

Each method in the MyTest class will be run as a test. Each test method may
contain zero or more assertions. If all assertions pass, the test passes. If any
assertion fails the whole test method immediately aborts and is marked as having
failed. Unlike Test::More above, we don't need to explicitly count the number
of tests for `plan()`. Instead, the library counts the number of methods for
us automatically.

Rosella's MockObject library can be used together with the Test library to add
MockObject support to your tests. Mock Objects, as I've said before and I'll say
a million times again in the future, are tools that can do as much harm as good,
especially if they are used incorrectly. Here's an example of a test using
MockObject:

    $include "rosella/test.winxed";
    $include "rosella/mockobject.winxed";
    class MyTest {
        function test_one() {
            var c = Rosella.MockObject.default_mock_factory()
                .create_typed(class MyTargetClass);
            c.expect_method("foo")
                .once()
                .with_args(1, 2, "test")
                .will_return("foobar");
            var o = c.mock();
            var result = o.foo(1, 2, "test");
            self.assert.equal(result, "foobar");
            c.verify();
        }
    }
    function main[main]() { Rosella.Test.test(class MyTest); }

You have to do more setup for the test with mockobjects, but you get a lot
more flexibility to do black-box testing and unit testing with proper
component isolation. I won't try to sell mock object testing here in this post,
only demonstrating that it is both possible and easy with Rosella.

Parrot has several thousand tests in its suite. Winxed has a small but growing
test suite. Rosella currently runs "728 tests in 116 files". parrot-libgit2 has
a growing test suite. Rakudo has a gigantic spectest suite. Jaesop has a growing
suite of tests. NQP has tests. PACT is going to have extensive tests, once it
has code worth testing. Plumage has tests (though not nearly enough!). PLA has
a relatively large suite of tests. Testing is a hugely important part of the
Parrot ecosystem, and we currently have several tools to help with testing.
Expect the trend to continue, with more tests being written for more projects
in 2012.


