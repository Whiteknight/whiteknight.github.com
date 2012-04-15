---
layout: post
categories: [Parrot, ParrotStore]
title: ParrotStore
---

I created a new repo for a new project: ParrotStore. ParrotStore intends to
provide some storage and persistance (and caching and database) solutions for
Parrot. At the time of writing this post we have two: Memcached and MySQL. A
third, MongoDB, is on the way.

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
I have things mostly set up, but am running into errors linking to the dynamic
libraries for the C client driver. Once I get the binding issues set up, I've
got a few bits of code ready which *should* work, after maybe a small amount of
effort:

1. I've got a MongoDbContext PMC which allows us to save BSON (the document
storage format used by MongoDB) to the database
2. I've got a BsonDocument PMC which represents a BSON document and allows you
to create one for saving to the DB.
3. I've got a tool, using Rosella's JSON library, to convert JSON to BSON.

I don't yet have a mechanism for reading documents back out of the DB, or
modifying documents in place.

Once I get my linking problems figured out, I'm going to get back to work on
these bindings and hopefully give Parrot a full-featured client interface to
MongoDB.

## Build System and Project Setup

ParrotStore contains a bunch of sub-projects which are really only related by
theme. They're all solutions for storing stuff, but they don't really relate to
each other besides that. So, the build system is set up to easily build these
projects individually. At the terminal, if you have `make`, you can build them
like this:

    make memcached
    make mysql
    make mongodb
    make            # attempts to build them all

This is great for if you don't have the mysql or mongodb development packages
installed but you want to get the memcached library (or any other combination).

Internally, the makefile calls a distutils-based `setup.winxed` program for
building the various components, but you shouldn't use `setup.winxed` directly.

Like Rosella, which is a prerequisite for this project, ParrotStore will be a
collection of things not one big monolithic system. It will provide a
Memcached interface in one standalone library, a MySQL interface in one, a
MongoDB interfae in one, and other interfaces separately too. Some of them
(like Memcached) will be pure parrot. Other things like MongoDB will have
C-level components too. Where Rosella has always promised to be pure Parrot,
ParrotStore cannot and should not follow such a rule.

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
this is all so new and experimental. I need to add a test suite

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



