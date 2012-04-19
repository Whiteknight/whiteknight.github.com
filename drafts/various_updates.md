---
layout: post
categories: [Parrot, Rosella, ParrotStore]
title: Various Updates
---

Here are some updates on various projects I've been working on:

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
current master I gave it a spin and most things looked good. However several
tests fail, mostly tests specifically for Eval PMC or tests which incorrectly
rely on its idiosyncratic behavior. I'm going to spend a little bit of time
tonight or this weekend to fix these tests, and hopefully get this branch
mergable soon.

## Sub Flags Cleanup

My `remove_sub_flags` branch, tasked with removing the old `:load` and `:init`
flags from Parrot and replacing them with the new `:tag()` syntax is right where
I left it a few weeks ago. I'm down to a relatively small list of test failures,
the solution to most of which is to update the syntax in the tests themselves.
A handful of tests such as those using the `parrot-nqp` and `winxed` compilers
are failing because I need to update those compilers first to generate the
correct code so the tests can run correctly.

I *think* I'm in the home stretch with that branch. I would like to get it
stable enough for wide-spread testing sometime this month. It represents such a
big and jarring change that maybe we want to hold it on the queue until after
the supported 4.4 release or even later.

## PCC

Bacek has been doing a lot of refactoring in PCC land, trying to fix some
slow and infelicitious aspects of it. I've gotten a set of new PCC-related
opcodes added to core and have a few more that I want to add, including new
variants of `set_args`, `get_params` and friends to take explicit context
arguments instead of using magical behavior to try and find them automatically.
A few patches to IMCC and the new behavior might go in without anybody noticing.

## Rosella

Rosella is mostly where I want it to be right now. I'm planning to change around
the development cycle to stick to supported releases of Parrot and Winxed
instead of tracking HEAD for both of them. I'm going to promote one or two more
libraries to "stable" status and then put out a release sometime after Parrot
4.4 hits the news stands next month.

## Google Summer of Code

GSOC is keeping me pretty busy so far.
