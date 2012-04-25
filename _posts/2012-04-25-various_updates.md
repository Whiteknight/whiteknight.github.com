---
layout: post
categories: [Parrot, Rosella, ParrotStore]
title: Various Updates
---

Here are some updates on various projects I've been working on or been planning
to work on:

## ParrotStore

In my post introducing ParrotStore, I mentioned that I only had support for
MySQL, Memcached, and a little bit of stuff working for MongoDB. In the past
few days I've also added SQLite3 support. Now you can do this, after installing
the prequisites, building and installing:

    var sqlite3_lib = loadlib('sqlite3_group');
    var sqlite3 = new 'SQLite3DbContext';
    sqlite3.open("test.sqlite3");
    sqlite3.query("INSERT INTO tbl1 (name, number) VALUES ('Andrew', 100)");
    var result = sqlite3.query("SELECT * FROM tbl 1");
    for (var row in result) {
        for (string colname in row)
            print(colname + "=" + string(row[colname]) + " ");
        say("");
    }

SQLite3 offers a bunch of features that I don't tap into yet, but we have a good
start and can do some basic work with it already.

Also, I mentioned that we didn't support queries with multiple result sets in
the MySQL bindings. Well, now we do (and we do in SQLite3 too):

    var result1 = mysql.query("CALL my_stored_proc");
    var result2 = sqlite3.query("SELECT * FROM tbl1 ; SELECT * from tbl2");

If the query returns one result set, a DataTable object is returned. If it has
multple result sets, an array of DataTables is returned instead.

## Eval PMC

I went digging through my backlog of old branches last night and found my
incomplete branch for removing the deprecated Eval PMC. After updating to
current master I gave it a spin and most things looked good. I fixed all the
core parrot tests and then moved on to the rest of the ecosystem.

Winxed works fine with the PackfileView PMC instead of the Eval PMC. I made a
few of those updates in the past, so it mostly worked out of the gate. Rosella
compiled and ran like a charm too.

NQP-rx works fine because it mostly relies on the PCT libraries that ship with
Parrot, and which I had already fixed.

The new NQP is a little bit more of a hassle. It took me a little bit of effort
to figure out the bootstrapping mechanism, but after a few hours of hacking I
had NQP building on the new Parrot using PackfileView instead of Eval. However,
one of the regex tests hangs indefinitely now and I'm having trouble tracking
that down. this project may get bumped down to a lower priority level until I
can either figure out what the problem with NQP is, or until I can enlist some
help to fix it.

I would like to merge this branch as soon as NQP is fixed and I can prove that
I can build it and Rakudo on the branch.

## Sub Flags Cleanup

My `remove_sub_flags` branch, tasked with removing the old `:load` and `:init`
flags from Parrot and replacing them with the new `:tag()` syntax is right where
I left it a few weeks ago. I'm down to a relatively small list of test failures,
the solution to most of which is to update the syntax in the tests themselves.
A handful of tests such as those using the `parrot-nqp` and `winxed` compilers
are failing because I need to update those compilers first to generate the
correct code so the tests can run correctly.

After fixing NQP-rx and Winxed, I need to get started testing out the new NQP
and Rakudo. I suspect both of those two things will be made to work without too
much effort.

It turns out that the Eval PMC deprecation work overlaps with this slightly, so
the things I change for that branch should help reduce failures in this branch
too. After I get Eval deprecated and removed, I'll come back to this branch and
see where things stand.

This is such a large and disruptive change that I can't imagine we would want
a merge before the 4.4 release, even if I got all the bugs ironed out. We could
be a month or more away from a merge, so I'm not listing this work as high
priority.

## PCC

Bacek has been doing a lot of refactoring in PCC land, trying to fix some
slow and infelicitious aspects of it. I've gotten a set of new PCC-related
opcodes added to core and have a few more that I want to add, including new
variants of `set_args`, `get_params` and friends to take explicit context
arguments instead of using magical behavior to try and find them automatically.
A few patches to IMCC and the new behavior might go in without anybody noticing.
I've talked more about this in past posts, and I'm sure I'll have more to say
when I start making changes.

## Rosella

Rosella is mostly where I want it to be right now. I'm planning to change around
the development cycle to stick to supported releases of Parrot and Winxed
instead of tracking HEAD for both of them. I'm going to promote one or two more
libraries to "stable" status and then put out a release sometime after Parrot
4.4 hits the news stands next month. I've already promoted the Parse and Json
libraries to stable status. I will probably promote Xml and Net too, since I am
pretty happy with both of those two libraries and feel that they are almost
ready for general use.

After that, I suspect Rosella is going to take a back seat for a while, so I can
focus on some other projects.

## Google Summer of Code

GSOC is keeping me pretty busy so far. We accepted 4 projects this summer. The
fifth project, which was to do some work on the Jaesop Stage 1 compiler, was
lost because the student was accepted to a different organization instead. The
four remaining projects are:

1. **Security Sandbox** by Justin
2. **Mod_Parrot 2.0** by brrt
3. **LAPACK Bindings** by jashwanth
4. **PACT Assembly** by benabik

I think these projects will be very cool, and I am looking forward to see what
kinds of great code they can produce this summer.

## Green Threads

nine has been doing some amazing work on his threading branch. Yesterday he
informed me that he had a solution to make green threads work on Windows, and
had already implemented part of it. That's awesome, because I was planning to
work on porting the green threads to windows next, but if he's doing it then I
don't have to.

Some of the performance numbers he's been getting are pretty impressive for
certain tasks. Some benchmarks he has are even showing a significant threading
performance improvement over a similar benchmark written in perl5.

I've been doing some testing on his branch and things are looking mostly good
except for one or two remaining GC-related bugs that need to be ironed out.
After that, if we can get some concensus, I would love to start talking a merger
shortly after 4.4.

## 6Model

With Green Threads possibly off my TO-DO list, Eval PMC Deprecation mostly
wrapped up and remove_sub_flags on the back burner, I can start moving towards
my next project: 6model. And I can do it much earlier than I was expecting. I'm
going to mine benabik's rejected 6model project proposal for some ideas, then
I'm going to jump in and try to get things working. I suspect things could get
moving pretty quickly, if I can keep my level of free time relatiely high.

