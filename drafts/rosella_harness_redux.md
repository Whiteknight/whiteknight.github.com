---
layout: post
categories: [Parrot, Rosella, Harness]
title: Rosella Harness Library Redux
---

My last post about Rosella talked about a major refactor of the Rosella Test
library. That refactor brought some backwards-incompatible changes to the
library, although I think the net result is that the system is more
usable, more configurable, and much better designed.

Recently I've finished another major refactor, this time for the other side
of the test system: the TAP Harness library. These recent changes are also
significant and backwards-incompatible with the earlier versions of the code,
but again I think the final result is well worth the effort.

## Overview

As I've written about in previous posts, writing a new test harness with the
Rosella Harness library used to be quite easy. In fact, I was able to write
a very general purpose harness in 6 lines of NQP code:

    INIT { pir::load_bytecode("rosella/tap_harness.pbc"); }
    my $harness := Rosella::build(Rosella::Harness);
    $harness.add_test_dirs("NQP", 't', :recurse(1));
    $harness.run();
    my $output := Rosella::build(Rosella::Harness::Output::Console);
    $output.show_results_summary($harness);

This is a pretty decent system, although it isn't too configurable. Nowhere
along the line am I able to effectively subclass any of the key components
without having to subclass everything and effectively completely rewrite the
whole library.

The new version of this is more verbose, but with the benefit of having much
better opportunity for customization:

    INIT { pir::load_bytecode("rosella/tap_harness.pbc"); }

    my $factory := Rosella::build(Rosella::Harness::TestRunFactory);
    $factory.add_test_dirs("NQP", "t", :recurse(1));
    my $testrun := $factory.create();
    my $harness := Rosella::build(Rosella::Harness);
    my $testview := $harness.default_view();
    $testview.add_run($testrun, 0);
    $harness.run($corerun, $testview);
    $testview.show_results();

The old version was 6 lines of code, the equivalent new version is 9 lines.
This isn't a huge extra expenditure of programmer effort, but as I mentioned
before, I more than make up for it in terms of new features.

## Model-View-Controller

The new Harness architecture is based on a proper [MVC][] design. I don't know
why the older version of it wasn't because a TAP harness really lends itself
well to the MVC architecture. In MVC we have three primary components: the
Model (`Rosella::Harness::TestRun`, and friends), the View
(`Rosella::Haress::View`), and the Controller (`Rosella::Harness`).

[MVC]: http://en.wikipedia.org/wiki/Model%E2%80%93View%E2%80%93Controller

When we're running a harness, the first thing we do is put together a TestRun
object. To do this, we use the new `Rosella::Harness::TestRunFactory`. We load
in various tests either by file name or by directory name (including recursive
directory searching). Actually, we can create several individual TestRun
objects and run them independently if we want to.

After we have a TestRun or even several TestRuns, the next steps are to create
a Harness and a View. Above, we just use a default View instance,
(`$harness.default_view()`), but you can easily provide your own custom
subclass or stand-in instead. This is a great option if you want to completely
change the way status is reported.

What we get from this architecture is great separation of concerns. The
Harness only needs to worry about running tests. The View only needs to worry
about displaying information to the user. The TestRun Model has all the logic
for organizing tests and keeping track of statistics and results. Most
importantly, the Harness and the TestRun don't need to contain any logic for
how to display information to the user. The TestRun doesn't need to worry
about how to run tests. The View doesn't need to know anything about how the
tests are run, how the statistics are kept, or where the tests come from in
the first place.

## Future

This library is in the basic shape it will take for the first Rosella release
sometime late next month (after Parrot 3.3). There are several improvements I
have planned to make at some point, most importantly is rewriting the TAP
parser to be less homebrew. If anybody is interested in using this library,
let me know what kinds of features you would like to see added to it before
the release.

