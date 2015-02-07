---
layout: post
categories: [Parrot, Rosella, Harness, Query]
title: Rosella Harness and Query Improvements
---

In my work on [Jaesop][], I realized that some parts of the Rosella Harness
library were a little bit more messy than I would like. I decided to take some
time and get that library raised up to a better level of quality. To make some
cleanups, I used the Query and FileSystem libraries for certain tasks, which
turned out to be a great move, because I identified nifty new features that
were needed in those libraries as well.

[Jaesop]: http://github.com/Whiteknight/jaesop

### Query Streams

The first version of the Query library functionality was very straight
forward. It basically provided the implementations of some higher-order
functions and method semantics that allowed calls to be chained together. Here
is a quick example:

    var result = Rosella.Query.as_queryable([1, 2, 3, 4])
                    .filter(function(i) { return i % 2 == 0; })
                    .map(function(i) { return i * 2; })
                    .fold(function(s, i) { return s + i; }, 0)
                    .data();

That example, clearly contrived, takes an array of numbers. It filters out the
odd numbers, then multiplies everything else by 2 and sums them together. It's
simple and straight-forward: The `filter` takes an input array and generates
an output array of values which meet the requirements. The `map` routine takes
an input array and produces an output array. The `fold` routine takes an
input array and outputs a single integer number. If each method output its
name to the console when it was invoked, we would see something like this:

    filter filter filter filter
    map map
    fold fold
    data

We do all the filtering first, then all the mapping, then all the folding.
It's very straight forward, but it's also eager, which isn't great when we
would rather be working with a lazy object.

The new addition to Query is the **Stream**. A Stream is any iterable object
which might prefer to be read lazily. Here's an example that I'm playing with,
using some new improvements to the FileSystem library as well:

    var f = new Rosella.FileSystem.File("foo.txt");
    var result = Rosella.Query.as_stream(f)
                    .take(5)
                    .filter(function(l) { return l != null && length(l) > 0; })
                    .map(function(l) { return "<<" + string(l) + ">>")
                    .data();

I've updated `Rosella.FileSystem.File` to be iterable. The default File
iterator reads the file line by line. This is a new feature and isn't really
configurable yet. In this example, we create a Stream from the File object.
We `take` the top 5 lines from the file, remove any empty lines, and surround
the remainder in `<< >>` brackets. The best part is that we read the file
lazily. This example *only* reads the top 5 lines of the file, it does not
read the entire text of it. That's a big help if we have a huge file, or if
we have something like a long-lived pipe that is spitting out an endless
sequence of data. Another thing that is different about Streams is that they
are interleaved. To see what I mean, if the methods above printed out their
names when invoked, we would see this pattern:

    take filter map
    take filter map
    take filter map
    take filter map
    take filter map
    data

Where the first example did all the `map`s first and all the `filters` second,
this example does them each one at a time for each input data item. Some of
the methods which are on a normal Queryable aren't present on Stream, and some
of the Stream methods aren't lazy. Some of them need to be eager, like
`.data()`, `.sort()` or `.fold()`.

If you want an updated harness for your project, it's easy to get one. Even
easier than copy+pasting the code from above. If you have Rosella installed,
you can automatically create a harness from a template. run this command at
your terminal:

    rosella_test_template harness winxed t/ > t/harness

That's all you need, and you'll get a spiffy full-featured harness which takes
advantage of all the new features I've been working on. If you prefer your
harness be written in NQP instead of Winxed, just change the "winxed" argument
above to "nqp" and you get that.

I'm going to work on an installable harness binary so you can just use one
without needing to create your own harness. I don't have it yet, but it will
not be too hard to make.

### Harness Cleanups

The harness is basically a huge iterator. You set up a bunch of tests
organized into a list of test runs. The harness iterates over each test run,
iterates over each test, gets the output, and iterates over the lines of text
in that output to get results. Then it iterates over all test runs, iterates
over all result objects to get the results display to show to the user. This
sounds like a perfect use for Query functionality, doesn't it? That's exactly
what I thought, anyway. I reimplemented several parts of it using Query and
the new Stream object. The input is set up as a stream over a pipe, and the
TAP parsing is implemented as a stream of tokens from a String.Tokenizer.
Combine those changes with some refactors, fixing abstraction boundaries, and
an eye towards test coverage, and the new code is much prettier than the old
code.

What most affects users is that harness code can now be cleaner. Here is what
a simple harness used to look like in Winxed:

    function main[main]() {
        var rosella = load_packfile("rosella/core.pbc");
        using Rosella.initialize_rosella;
        initialize_rosella("harness");
        var factory = new Rosella.Harness.TestRun.Factory();
        var harness = new Rosella.Harness();
        var view = harness.default_view();
        factory.add_test_dirs("Winxed", "t", 1:[named("recurse")]);
        var testrun = factory.create();
        view.add_run(testrun, 0);
        harness.run(testrun, view);
        view.show_results();
    }

Here is what a new one looks like:

    function main[main]() {
        var rosella = load_packfile("rosella/core.pbc");
        var (Rosella.initialize_rosella)("harness");
        var harness = new Rosella.Harness();
        harness.add_test_dirs("Automatic", "t", 1:[named("recurse")])
            .setup_test_run(1:[named("sort")])
        harness.run();
        harness.show_results();
    }

Not too bad for 6 real lines of code! The new `"Automatic"` test type reads
the she-bang line ("`#! ...`") from the test file to determine how to execute
it. If you want to specify a particular language like `"NQP"` or `"Winxed"`,
you can still do that too. Notice also that we can sort files by filename too,
if we pass in that parameter to `.setup_test_run`. At the moment, sorting is
only alphabetic and per-run only. We don't shuffle files between runs.

Harnesses using the older-style should all still work. I've tried as well as
I can to keep the code backwards compatible. If you have a harness that
doesn't work anymore after updating Rosella it's a bug and I would love to
hear about it. Also, all the same capabilities are there: The ability to
substitute a custom View, the ability to break files up arbitrarily into test
runs, the ability to specify custom subclasses of TestRun or TestFile or other
stuff like that too, if you need some custom semantics.

After these rewrites, the version of the Harness library is 3. I don't know if
anybody follows along with these per-library version numbers, but it is a
decent point of reference. I don't expect to be making any large changes to
this library again for a while.

### File Iterators

I've implemented a very quick and basic iteration facility for files as part
of the Rosella FileSystem library. The iterator type I have so far is a basic
line iterator, calling the `.readline()` method on the given handle until EOF.

There are two ways to use the new FileIterator class: Iterate over a Rosella
File object directly, or create an IterableHandle object over an existing
low-level handle.

    // Iterate over a File object
    var file = new Rosella.FileSystem.File("foo/bar.txt");
    for (string line in file) {
        ...
    }

    // Make an Iterable Handle
    var fh = new 'FileHandle';
    fh.open("foo/bar.txt", "r");
    var ih = new Rosella.FileSystem.IterableHandle(fh);
    for (string line in ih) {
        ...
    }

That second option is actually kind of neat, because you can use it over any
Handle object: FileHandle (including standard input and pipes),
StringHandle, Socket, etc.

If you really want to be tricky, you can do what I do in the Harness library
and create a stream over a handle and really do a lot of cool stuff:

    var fh = new 'FileHandle';
    fh.open("foo/bar.txt", "r");
    var ih = new Rosella.FileSystem.IterableHandle(fh);
    var stream = Rosella.Query.as_stream(ih);
    stream
        .take(5)
        .filter(function(l) { return l != null && length(l) > 0 && substr(l, 0, 1) != "#"; })
        .project(function(l) { return split(";"); })
        .foreach(function(string s) { say("Look at this: " + s); })
        .execute();

Again, this is a contrived example, but it should become apparent what kinds
of stuff you can do with this.

I don't have directory iterators yet. That is something I haven't needed yet,
but for which I can see some uses.

### String Tokenizer Iterators

I don't know why I didn't think about it earlier, but now Tokenizers are
iterable as well. If you have a tokenizer, you can iterate over them in two
different ways:

    var tokenizer = new Rosella.String.Tokenizer.Delimiter(",");
    tokenizer.add_data("a,b,c,d,e,f");
    for (string field in tokenizer) {
        ...
    }

    tokenizer.add_data("g,h,i,j,k,l");
    for (var t in tokenizer) {
        ...
    }

In the first, we do a shift_string operation which returns the raw string
data. In the second we do a shift_pmc operation which returns the Token
object. The Token contains some information like the type of token, some
custom metadata, etc.

And of course, since you can iterate over them, you can use a Stream:

    var tokenizer = new Rosella.String.Tokenizer.Delimiter(",");
    tokenizer.add_data("a,b,c,d,e,f");
    var stream = Rosella.Query.as_stream(tokenizer);
    var new_data = stream
        .map(function(t) { return t.data(); })
        .filter(function(s) { return s != "b" && s != "e"; })
        .fold(function(a, b) { return sprintf("%s,%s", [a, b]); })
        .next();

### Upcoming Changes

The new Harness library is making me pretty happy. I'm working on tests now,
because the library has never had good test coverage and is suddenly much more
testable than it ever has been. Considering Harness is a central part of my
TAP testing strategy, it's kind of embarrassing that it has never been well
tested itself. I am working on test coverage in a branch, and will probably
be merging that to master soon. I don't expect to make any big changes to the
library for a while after that.

The Stream class is not well fleshed-out yet and is absolutely untested
besides the tests for its use in Harness. I need to finish up with a few of
the features that I haven't needed yet, and then test it all. There are a few
tweaks I might want to make to the way it works, but for the most part I am
pretty happy with how it turned out and how fun it is to use.

The FileIterator and Token Iterator types I mentioned are even newer and less
mature than Stream is, and need some serious review. They've been useful tools
to get me to this point, but they can definitely stand to be improved in
non-trivial ways. I've got some big refactors of the String library planned
in the future, so if anybody has any requests for features now is a good time
to mention them.

In my testing of Harness, even though it's not complete yet, I've already
found a few changes that need to be made in the MockObject and Proxy libraries
as well. I plan to take a good hard look at those things to make sure they
are up to the level I expect them to be. Also, I have a few other unstable
libraries floating around that need attention, and could potentially become
stable if I like where they are going.

