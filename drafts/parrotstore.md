---
layout: post
categories: [Parrot, ParrotStore]
title: ParrotStore
---

I created a new repo for a new project: ParrotStore. ParrotStore intends to
provide some storage and persistance (and caching and database) solutions for
Parrot.

## Memcached

The first thing I've written is a rudimentary interface to Memcached for
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

Memcached is easy, so I started working on it first. Other projects won't be
quite so easy.

## Future Projects

The goal of ParrotStore is simple persistance. In a sense it might become
something like an ORM, mapping Parrot data to and from various persistance
mechanisms. This project does not intend to do any embedding, whether Parrot
embedded in a database or a database embedded in Parrot, or whatever else. The
Database (or cache or whatever) is separate, and ParrotStore just provides an
interface to it. For instance, the PL/Parrot project embeds Parrot into the
Postgres DB. ParrotStore would provide an external interface for querying it.

Two other things I want to add in the future are interfaces for MySQL and
MongoDB. These things are probably going to want to be implemented at the C
level through wrapper PMCs, but I haven't settled on a design yet. Once we have
these things and some kind of presence on the web (a revamped ModParrot or
FastCGI bindings seem like good options right now), creating real websites
running on Parrot starts to become a reality.

I also want to add a custom caching mechanism for storing frozen PMCs to file
and fetching them again. Multiple backends to a PMC mechanism would allow us
to store PMCs to various persistance systems for later use. This is another
thing that I've wanted for a while, but I haven't quite nailed down a design
yet.

## ParrotStore Architecture

Like Rosella, which is a prerequisite for this project, ParrotStore will be a
collection of things, not one big monolithic system. It will provide a
Memcached interface in one standalone library, and will provide other things as
separate libraries too. Some of them (like Memcached) will be pure parrot. Other
things like MongoDB might have C-level components too. Where Rosella has always
promised to be pure Parrot, ParrotStore cannot and should not follow such a
rule.

The first thing I need to do now is set up a proper build system and a test
suite. Then I want to start adding new things. It's actually exciting to think
about all the cool new things we'll

