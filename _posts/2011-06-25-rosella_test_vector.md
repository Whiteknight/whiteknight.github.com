---
layout: post
categories: [Parrot, Rosella, Test]
title: Vectorized Tests With Rosella
---

A few days ago I was talking to Parrot hacker and GSoC student bubaflub about
his GSoC project. For people not taking detailed notes, bubaflub is writing
some improved bindings to the GMP library for Parrot, to bring real support
for arbitrary-precision mathematics. It's a cool project, and could have some
real benefits for people writing certain types of applications on Parrot.

Anyway, in testing arithmetic operations he was running into a lot of very
repetitive test code. He wanted something that would help automate some of
his tests, or at least make them less verbose to write in large quantities.

I got an idea in my head to write up some functionality for vectorized tests.
Vectorized tests are tests where there is a single test function which can be
executed on each item in an input vector. I wrote up some prototypes, played
around with some ideas, and finally pushed some new code to do the job.

So, we now have vectorized tests in Rosella. Here's an example of their use in
Winxed code:

    function main[main]()
    {
        using Rosella.Test.test_vector;
        test_vector(function(var self, var data) {
            self.assert.equal(data[0] + data[1], data[2]);
        }, [
            [1, 2, 3],
            [3, 4, 7],
            [7, 8, 15]
        ]);
    }

In this example, I have one test method which is invoked as three separate
tests, one for each item in the data array. Running that program gives the
following TAP output:

    1..3
    ok 1 - test 1
    ok 2 - test 2
    ok 3 - test 3
    # You passed all 3 tests

So that's what we expect, and all is right with the universe. Instead of an
array of data, we can use a hash to give names to all the tests:

    function main[main]()
    {
        using Rosella.Test.test_vector;
        test_vector(function(var self, var data) {
            self.assert.equal(data[0] + data[1], data[2]);
        }, {
            "1 + 2 = 3"  : [1, 2, 3],
            "3 + 4 = 7"  : [3, 4, 7],
            "7 + 8 = 15" : [7, 8, 15]
        ]);
    }

...And the output:

    1..3
    ok 1 - 1 + 2 = 3
    ok 2 - 3 + 4 = 7
    ok 3 - 7 + 8 = 15
    # You passed all 3 tests

In the older-style tests with `Rosella.Test.test`, you had many functions and
a single shared data item, the `TestContext` object. `test_vector` is exactly
the opposite: a single shared code item and a series of data points. Both have
their uses in different types of situations, and now you have the option
when you are writing your tests.

At the moment you cannot combine the two forms. However, I am planning to
upgrade Rosella in the near-to-medium future to support nested TAP output, so
we can run multiple test suites in a single file. I'm not sure what the
interface for such a thing will look like (suggestions welcome!), so I'm not
planning to start on that work quite yet. I'm also looking at functionality
to create multiple Suite objects, then merge them together into the final test
run. That may be what the nested TAP interface looks like or it may be
completely flat. I don't know yet.
