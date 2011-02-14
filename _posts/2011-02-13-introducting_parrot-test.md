---
layout: post
categories: [Parrot, Parrot-Test]
title: Introducting Parrot-Test
---

Parrot-Linear-Algebra, my linear algebra library for Parrot, had been relying
on the [Kakapo][] library for several months to run its test suite. Kakapo was
a really great framework and, most importantly for my purposes, it provided a
pretty great xUnit-style testing framework.

[Kakapo]: http://code.google.com/p/kakapo-parrot

Kakapo was created and maintained by Austin Hastings, and when he became busy
and was unable to maintain it, Kakapo went unmaintained. I had made a few
fixes over time to keep PLA running, but I was only able to provide fixes for
the things that PLA directly relied on. Much of Kakapo has not worked for
months, including its own test suite. I kept the parts I needed alive in a
fork on github, but I couldn't keep up with maintaining even that.

I decided that I wanted to work on a different testing solution. I didn't want
to rely on a library that wasn't actively maintained. Kakapo was great and I
loved the test functionality it provided, but if it didn't work it was useless
to me. Today, I started a new project to provide the same kinds of features
in a way that I could manage and keep up with the maintenance for.

Today, I'm happy to introduce a new project, [Parrot-Test][]. Parrot-Test is
a test-focused framework library that aims to provide the tools and utilities
that other projects would need to implement their own test suites. Currently
Parrot-Test only provides a port of Kakapo's unit tests, but I plan to expand
its functionality significantly in the coming months.

[Parrot-Test]: http://github.com/Whiteknight/parrot-test

As of this evening, the PLA test suite has been updated to use Parrot-Test.
Suddenly all the tests are running and passing again, which they haven't done
for a few months now. With the tests running again I'm preparing to cut a new
release of PLA. I don't know when that will be, but at least now I know that
it is *possible* to do. That's quite a good thing to know.

Right now Parrot-Test only contains the UnitTest features from Kakapo, with
some changes, omissions, and fixes. There are several things I would like to
add in the future:

1. Libraries for building test harnesses, including the ability to submit
   smoke reports.
2. A mock object library. I may borrow ideas from Kakapo's Cuculinae library,
   or I may try to brew my own.
3. Implementations and some reimplementations of some TAP libraries, for
   people who like Test::More over xUnit.

By keeping the libraries in this project more modular and separate, I think
I can provide a lot of great and helpful features for people to mix-and-match
as necessary.

So how do we get started using Parrot-Test? From an NQP program, we start by
including the necessary bytecodes:

    INIT {
        pir::load_bytecode("parrot_test_xunit.pbc");
    }

Next, we define a new test class inheriting from `UnitTest::Testcase`:

    class MyTest is UnitTest::Testcase {
    }

Inside that class, any methods which start with "`test_`" are treated as
tests and automatically executed. Test methods execute and, if nothing goes
wrong, are marked as success. That's really the gist of it. You have a test
method, you execute some code, and if nothing goes wrong it's treated as a
success. An unhandled exception would cause the test to be treated as a
failure. Also, an assertion failure would cause the test to fail.

Assertions are special functions, like those in `Test::More` which test a
condition. If the condition is true, nothing happen. If the condition is false
it throws a UnitTestFailure exception and the test is treated as a failure.

    class MyTest is UnitTest::Testcase {
        method test_empty() {
            # Do nothing. Nothing happens. Nothing goes wrong. Success.
        }

        method test_assert() {
            Assert::equal(1, 1, "Oops!");
            # test that the two values are equal (they are). If the test
            # fails, the message argument is displayed
            # This test passes.
        }

        method test_fails() {
            Assert::not_equal(1, 1, "what?);
            # Failed assertion, failed test
        }

        method test_many() {
            Assert::equal("a", "a", "a != a");
            Assert::not_equal("b", "c", "b == c");
            Assert::equal("d", "e");   # Oops!
            # You can have many assertions in a single test. Assertions are
            # not tests themselves, so adding them doesn't increase your test
            # count. If any assertion in a test fails, the entire test fails
        }
    }

Finally, to run your test you create an object of your test class, create a
suite for it, and run the suite:

    my $test := MyTest.new;
    $test.suite.run;

Right now we only handle one test class per file, although I'm working to
remove that restriction.

So there's a quick primer for how to use Parrot-Test in your project. I have
many other things that I want to add to this project in the future, and I'm
sure there are things I haven't thought about yet. If you have any questions,
suggestions, or ideas, please let me know so I can include them in the
development plan. If you want to get involved and start hacking on this
project you can create a fork or ask for a commit bit. Either way is fine with
me.
