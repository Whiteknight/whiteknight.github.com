---
layout: post
categories: [Parrot, Rosella, Testing]
title: Unit Testing with Rosella
---

Let's say we have a new software project, and we want to have a suite of unit
tests for it. It's easy to put together a nice suite of tests for your project
with [Rosella][]. Here's how.

[Rosella]: http://github.com/Whiteknight/Rosella

## Basic Harness

First thing, let's put together a basic test harness. With Rosella's
`tap_harness` library this is extremely easy. Let's put together a very basic
implementation first:

    INIT { pir::load_bytecode("rosella/tap_harness.pbc"); }
    my $harness := Rosella::build(Rosella::Harness);
    $harness.add_test_dirs("NQP", 't', :recurse(1));
    $harness.run();
    my $output := Rosella::build(Rosella::Harness::Output::Console);
    $output.show_results_summary($harness);

Six lines of code later, we have a harness that automatically reads all tests
from the `t/` directory, searching recursively for files. We run the tests
and output a basic summary. Now we can get started with writing some tests,
and later we can make a more interesting harness.

## Basic First Tests

Now that we have a basic harness, we can start writing some basic tests. We're
going to use Rosella's `xunit` library. Here's our first test file, with some
example tests:

    INIT { pir::load_bytecode("rosella/xunit.pbc"); }
    Rosella::Testcase::test(MyFirstTest);

    class MyFirstTest is Rosella::Testcase {
        method test_First() {
            Assert::equal(0, 0, "Should be true!");
        }

        method test_Second() {
            Assert::equal(0, 1, "Ooops!");
        }

        method test_Third() {
            self.unimplemented("Haven't written this test yet!");
        }

        method test_Fourth() {
            self.todo("This is probably going to break...");
            Assert::equal(1, 2);
        }
    }

One test passes, one fails, one is listed as unimplemented, and one fails but
is marked "todo". We run this test file manually, and get the following
output:

    1..4
    ok 1 - test_First
    not ok 2 - test_Second
    # Ooops!
    not ok 3 - test_Third # TODO Haven't written this test yet!
    # Unimplemented: test_Third
    not ok 4 - test_Fourth # TODO This is probably going to break...
    # objects are not equal

And, if we run this in our basic harness, we get what we expect:

    t/test.t ... not ok (Failed 1 / 4)
    Result: FAILED
            Passed 3 tests in 1 files
            Failed 1 tests in 1 files

We're well on our way now. Let's beef up our harness a little bit next.

## Better Harness

Recursive searching for tests is nice, but Parrot's distutils library can hold
a list of tests and can feed it into our harness from the commandline. Let's
update the harness to support that.

    INIT { pir::load_bytecode("rosella/tap_harness.pbc"); }
    MAIN(pir::getinterp()[2]);

    sub MAIN(@ARGS) {
        pir::shift(@ARGS);          # Program name
        my $harness := Rosella::build(Rosella::Harness);
        $harness.add_test_files("NQP", |@ARGS);
        $harness.run();
        my $output := Rosella::build(Rosella::Harness::Output::Console);
        $output.show_results_summary($harness);
    }

Instead of specifying a list of directories to search, now we're specifying a
list of tests. We load them all into the harness, and we run it.

But, maybe that's not everything we want. We would like to run a quick sanity
test first, just to make sure that we can actually load our extension code
into Parrot. If we can't even do that, it doesn't make any sense to try and
run other tests. Plus, if we have a problem with something so fundamental and
it would cause all my tests to fail, we want to be able to pinpoint that
immediately. Now we are going to update the harness so that it runs a set of
sanity tests before running other tests:

    INIT { pir::load_bytecode("rosella/tap_harness.pbc"); }
    MAIN(pir::getinterp()[2]);

    sub MAIN(@ARGS) {
        pir::shift(@ARGS);
        my $harness := Rosella::build(Rosella::Harness);
        $harness.add_test_dirs("NQP", "t/sanity");
        $harness.run();
        if $harness.run_is_success() {
            $harness.add_test_files("NQP", |@ARGS);
            $harness.run();
        }
        my $output := Rosella::build(Rosella::Harness::Output::Console);
        $output.show_results_summary($harness);
    }

Now if the sanity tests fail we don't even bother to run any other tests.
Anything after the sanity tests would be guaranteed to fail anyway, so it's
no use running them when we know it's impossible. This saves us time and a lot
of error output in our console. Now with that out of the way, let's revisit
our earlier test.

## Better Tests

We saw the test output above, the names of the various test methods were
output to the console. That's nice, but maybe not as descriptive as we could
have. Now we can use the `self.verify()` method to give a more understandable
description to the tests:

    INIT { pir::load_bytecode("rosella/xunit.pbc"); }
    Rosella::Testcase::test(MyFirstTest);

    class MyFirstTest is Rosella::Testcase {
        method test_First() {
            self.verify("Test that 0 == 0. This is basic");
            Assert::equal(0, 0, "Should be true!");
        }
    }

Running this on the console gives us this better-looking output:

    1..4
    ok 1 - Test that 0 == 0. This is basic
    ...

Much nicer than what we had before.

## Better Harness Output

We would now like to add some improved error-reporting to the harness, so we
get a complete listing of troubled-tests. At the end of the harness, we add
these two lines:

    $output.show_error_detail($harness);
    $output.show_failure_detail($harness);

In this context, an "error" is a file that exited prematurely for some reason.
A "failure" is a test failure caused by an `Assert::` problem. In both cases,
we won't output anything if there are no tests in those categories. To show
an example of an "error", We're going to add a new test file which fails
parsing. Use your imagination to think of what such a file would look like.

Running the harness now gives us a list of tests that cause us problems:

    t/test.t .... not ok (Failed 2 / 5)
    t/test2.t ... not ok (aborted prematurely)
    # Test aborted with exit code 1
    Result: FAILED
            Passed 3 tests in 2 files
            Failed 1 files due to premature exit
            Failed 2 tests in 1 files

            List of failed tests by file:
                    test.t
                            test 2 - test_Second
                            test 5 - test_Five
            List of files with premature exits:
                    test2.t

The third and fourth tests from the `t/test.t` file were marked as TODO, so
they don't get included in the output lists. The `t/test2.t` file didn't
execute any tests. That file was a complete failure and gets handled
separately.

So there we have a new test harness and the beginnings of a new unit test
suite. All of this, and we haven't even scratched the surface of what
Rosella's test libraries are capable of doing. In future posts, I'll talk
about some of the more advanced features of these test libraries, and also
talk about some of the other cool Rosella libraries.

