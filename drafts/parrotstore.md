---
layout: post
categories: [Parrot, ParrotStore]
title: ParrotStore
---

I created a new repo for a new project: [ParrotStore](http://github.com/Whiteknight/ParrotStore).
ParrotStore intends to provide some storage and persistance (and caching and
database) solutions for Parrot. At the time of writing this post we have three
in development: Memcached, MySQL and MongoDB.

## Memcached

The first thing I wrote is a rudimentary pure-parrot interface to Memcached for
high speed caching. The interface looks like this:

    var memcached = new ParrotStore.Memcached(["192.168.1.1", "192.169.1.2"]);
    memcached.set("foo", "hello world!");
    :(int have, string content) = memcached.get("foo");
    say(content);

Or, if you want a simpler interface, you can do something like this:

    string content = memcached.autoget("foo",
        function() { return "hello world!"; }
    );

The `autoget` method will try to read from Memcached if the item exists, and
will invoke the callback to get the value otherwise (and save it to Memcached
for later use). Of course, for this to be practical the callback to generate
the content should be more expensive than a return of a constant string.

I havent't tested with multiple memcached servers yet, and I haven't implemented
several of the methods memcached supports. It's a start, however, and I can
already think of several potential uses for it.

## MySQL

MySQL is popular and extremely common, so I figured I should work on that next.
Plus, if we ever want to have a snowball's chance in hell of hosting a decent
PHP compiler, we're going to want easy and available bindings for MySQL. Now,
after a little bit of hacking today, we have it.

Here's what we can do in Parrot today:

    var lib = loadlib("mysql_group");
    var mysql = new 'MySQLDbContext';
    mysql.connect("localhost", "username", "password", "database", 0, 0);
    var result = mysql.query("DROP DATABASE foo;");
    say(result, " rows effected");      // "1 rows affected", if you had one

    result = mysql.query("SELECT * FROM bar");
    say(typeof(result));                // "MySqlDataTable"
    for (var row in result) {           // Iterate over all rows
        int idx = int(row);
        say("row " + string(idx));
        for (string column in row) {    // Iterate over all columns
            say(column + ": " + string(row[column]));
        }
    }

One thing I don't handle quite yet is handling multiple result sets. So if you
have a stored proc which returns multiple sets of data, you won't get any but
the first back into your program. I'll try to get that implemented as quickly
as I can.

## MongoDB

We're starting to use MongoDB at work, and I figured a great way to become more
familiar with this piece of software was to write bindings for it for Parrot.
Despite several unnecessary problems with linking to the Mongo C Driver
libraries, I've managed to produce a few results.

Mongo uses a storage format called BSON (similar to JSON), and stores BSON
documents as atomic units. ParrotStore implements a BsonDocument and a
MongoDbContext PMC type. As of this morning, you can create a BSON document and
insert it into the DB:

    var lib = loadlib("mongodb_group");
    var bsondoc = new 'BsonDocument';
    bsondoc.append_start_object("name");
    bsondoc.append_string("first", "Andrew");
    bsondoc.append_string("nick", "Whiteknight");
    bsondoc.append_end_obect();
    bsondoc.finish();

    var mongo = new 'MongoDbContext';
    mongo.connect("127.0.0.1", 27017);
    mongo.insert("local.foo", bsondoc);

The document is indeed written to the database, although I don't have any
methods yet to read it back out. The documentation for the C Driver for MongoDB
is lacking, but I have the source code handy and it is pretty readable. I hope
to have basic querying implemented by the end of the day.

Here are a few things I plan to add, either today or in the next few days:

1. Support simple querys and commands
2. Support introspecting and iterating over BSON documents
3. Implement a JSON->BSON translator (I have most of this written already).

There are several other features that I need to implement, although many of them
aren't necessary to say I have a minimally functional set: support for
replicated sets, support for atomic find/replace updates, support for cursors
and bson iterators, etc. There's a lot of work here, but I'm off to a pretty
good start already.

## Build System and Project Setup

ParrotStore contains a bunch of sub-projects which are really only related by
theme. They're all solutions for storing stuff, but they don't really relate to
each other besides that. So, the build system is set up to easily build these
projects individually. At the terminal, if you have `make`, you can build them
like this:

    make memcached
    make install_memcached
    make mysql
    make install_mysql
    make mongodb
    make install_mongodb
    make            # attempts to build them all
    make install    # attempts to build and install them all

This is great for if you don't have the mysql or mongodb development packages
installed but you want to get the memcached library (or any other combination).

Internally, the makefile calls a distutils-based `setup.winxed` program for
building the various components, but you shouldn't use `setup.winxed` directly.

Like Rosella, which is a prerequisite for this project, ParrotStore will be a
collection of things not one big monolithic system. It will provide a
Memcached interface in one standalone library, a MySQL interface in one, a
MongoDB interface in one, and other interfaces separately too. Some of them
(like Memcached) will be pure parrot. Other things like MongoDB will have
C-level components too. Where Rosella has always promised to be pure Parrot,
ParrotStore cannot and should not follow such a rule. Some things may turn out
to be implementable with NCI, but that's an experiment for later. Maybe, much
later.

Also, expect a lot of synergy between Rosella and ParrotStore. ParrotStore will
both use Rosella internally, provide many of the interfaces that other
Rosella-based projects expect, and add several extensions to make Rosella
features even more cool and powerful.

## Future Projects

The goal of ParrotStore is simple persistance. In a sense it might become
something like an ORM, or contain an ORM, mapping Parrot data to and from
various persistance mechanisms. This project does not intend to do any
embedding, whether Parrot embedded in a database or a database embedded in
Parrot, or whatever else. The Database (or cache or whatever) is separate, and
ParrotStore just provides a client interface to it. For instance, the PL/Parrot
project embeds Parrot into the Postgres DB. ParrotStore would provide an
external interface for querying it instead.

I do not yet have a runnable test suite. I've been doing ad hoc tests because
this is all so new and experimental. I need to add a test suite.

I also want to add a custom caching mechanism for storing frozen PMCs to file
and fetching them again. Multiple backends to a PMC mechanism would allow us
to store PMCs to various persistance systems for later use. This is another
thing that I've wanted for a while, but I haven't quite nailed down a design
yet.

I would like to add a client interface for Postgres. I suspect there are some
people floating around who could help make that a reality.

I think this project will probably grow organically, adding new storage backends
and cool interfaces for various purposes, and then adding some tools and
utilities that use these things. As with all my projects, feedback, requests,
suggestions, and questions about my basic compentency are always welcome.

