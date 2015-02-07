---
layout: post
categories: [Parrot, Rosella, Test]
title: Rosella Test Updates and Upgrades
---

The first half of this month was dominated with some epic illnesses between my
family members and myself, family functions and home maintenance. What little
spare time I've had otherwise has been devoted to writing code, as opposed to
writing blog posts about writing code. The blog has suffered.

The past couple days I've been working on Rosella's Test library. It's an old
but good library and is, as far as I am aware, the most full-featured and easy
to use testing tool in the Parrot ecosystem. With some of these most recent
changes the library is better still.

### Matchers

Kakapo had a series of Matcher routines and objects as part of it's testing
facilities, and for a long time I've been wanting to port some of those ideas
over to Rosella. As of last week, I have a simple version of them. Matchers
allow you to ask the question "is this thing like that thing", with a custom
set of rules. Let me give a basic example.

Previously in Rosella if you were unit testing a method which returned an array
and you wanted to check that the array contained the right values, you would
have to do something like this:

    var result = obj.my_method();
    var expected = [1, 2, 3, 4, 5];
    self.assert.is_true(result != null);
    self.assert.equal(elements(result), elements(expected));
    for (int i = 0; i < elements(expected); i++) {
        self.assert.equal(result[i], expected[i]);
    }

That's a lot of work, although you can cut it down a little bit if you know for
certain that the array isn't null. With the new matcher functionality, you
pass in two arrays and the Test library will match them for you:

    var result = obj.my_method();
    self.assert.is_match(result, [1, 2, 3, 4, 5]);

Internally the Test library maintains a list of matchers by name. When you pass
in two objects, it loops over the list looking for a matcher that can handle
the pair. In this case, one of the default matchers the library provides looks
for objects which implement the `"array"` role, and then does element-wise
matching on them. Another similar matcher does the same for hash-like objects
that implement the `"hash"` role.

Another matcher checks to see if one or both of the two objects are strings, and
then does a string comparison on them (converting the other, if it isn't a
string already) and the last of the default matchers is used to compare floating
point values with a certain error tolerance.

Since matchers are stored in a hash, you can access them by name, delete them,
add your own, and replace existing ones if you want new matching semantics. This
is especially useful in something like Parrot-Linear-Algebra, where I can say

    $!assert.is_match($matrix_a, $matrix_b);

...and the library will automatically compare the dimensions of the matrices and
the contents of them without needing nested loops and other distractions.

### Nested TAP

Another item I've had on my wishlist for a while now has been nested TAP. I've
always wanted to support it, and in theory at least the system was designed
modularly enough to generate it without too much hassle. Last weekend I put on
the finishing touches and now am proud to say that Rosella.Test can run nested
tests and generate nested TAP. At the moment the interface to use it is a little
ugly (I'm actively soliciting feedback!), but the capabilities are all there:

    function my_test_method()
    {
        self.status.suite().subtest(class MySubtestClass);
    }

    function my_vector_test_method()
    {
        self.status.suite().subtest_vector(
            function(var a, var b) { ... },
            [1, 2, 3, 4, 5]
        );
    }

    function my_list_test_method()
    {
        self.status.suite().subtest_list(
            function(var test) { ... },
            function(var test) { ... },
            function(var test) { ... }
        );
    }

Here's an example of output from a similar test file in the Rosella suite:

    1..4
        1..2
        ok 1 - test_1A
        ok 2 - test_1B
        # You passed all 2 subtests
    ok 1 - test_1
        1..3
        ok 1 - test 1
        ok 2 - test 2
        ok 3 - test 3
        # You passed all 3 subtests
    ok 2 - test_2
        1..5
        ok 1 - test 1
        ok 2 - test 2
        ok 3 - test 3
        ok 4 - test 4
        ok 5 - test 5
        # You passed all 5 subtests
    ok 3 - test_3
        1..1
        not ok 1 - test 1
        # failure
        # Called from 'fail' (rosella/test.winxed : 481)
        # Called from '' (t/winxed_test/Nested.t : 40)
        # Called from '' (rosella/test.winxed : 1589)
        # Looks like you failed 1 of 1 subtests run
    ok 4 - test_4
    # You passed all 4 tests

The fourth test expects a failure in the subtest, which is why it says it
passes when there is clearly some failure diagnostics appearing. This brings me
to my next point...

### Cleaner Diagnostics

Before when you ran a test and had a failure, you might see something like this:

    not ok 2 - ooopsie_doopsies
    # objects not equal '0' != '1'
    # Called from 'throw' (rosella/test.winxed : 851)
    # Called from 'internal_fail' (rosella/test.winxed : 1853)
    # Called from 'fail' (rosella/test.winxed : 481)
    # Called from 'equal' (rosella/test.winxed : 577)
    # Called from 'ooopsie_doopsies' (t/core/Error.t : 18)
    # Called from 'execute_test' (rosella/test.winxed : 1455)
    # Called from '__run_test' (rosella/test.winxed : 1483)
    # Called from 'run' (rosella/test.winxed : 1392)
    # Called from 'test' (rosella/test.winxed : 1747)
    # Called from '_block1000' (t/core/Error.t : 7)
    # Called from '_block1177' ( : 158)
    # Called from 'eval' ( : 151)
    # Called from 'evalfiles' ( : 0)
    # Called from 'command_line' ( : 0)
    # Called from 'main' ( : 1)
    # Called from '(entry)' ( : 0)

That's a huge mess, and it's a mess from two sides. At the top of the backtrace,
you see all sorts of Rosella internal functions involved in the assertion and
error handling. The bottom half of the backtrace is devoted to entry-way stuff.
In this case there's NQP-related entry code and then the Rosella entry code.
You, as the test writer, don't care about any of that. All you care about is the
code you wrote and where its broken. If you have to dig through a huge backtrace
to figure out where the error is, that's a big waste of time and effort.

Now, Rosella filters that crap out for you. Here's the same exact failure with
the new backtrace reporting:

    not ok 2 - ooopsie_doopsies
    # objects not equal '0' != '1'
    # Called from 'equal' (rosella/test.winxed : 577)
    # Called from 'ooopsie_doopsies' (t/core/Error.t : 18)

Here you see the important parts of the backtrace only: The parts you wrote and
the one assertion that failed. You don't see the internal garbage, you don't
see the entry-way garbage, because those things aren't of interest to the test
writer.

### Parrot-Linear-Algebra

Another small project I did a few days ago was getting the PLA test suite
working again. It's a testament to how stable both BLAS and Parrot's
extending interfaces are. Recent Rosella refactors removed some of the
special-purpose features that existed only for PLA and for no other reason
(and which were a pain in the butt to maintain). I fixed up the test suite and
PLA builds and runs perfectly now.

That's what I've been up to this month. I'm mostly done with my cleanups to the
Test library now, barring a few more interface improvements I want to make.
After that I've got a few projects to tackle inside libparrot itself. I'll write
more about those topics when I have something to say.
