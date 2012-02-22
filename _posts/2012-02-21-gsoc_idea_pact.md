---
layout: post
categories: [Parrot, GSOC2012]
title: Introspection, Disasembly and PACT (GSOC)
---

In [my post a few days ago I mentioned Google Summer of Code 2012][gsoc_post]
and gave a lightning list of simple project ideas that might be worth pursuing.
Today I'm going to expand on one of these ideas because it's fertile ground for
many possible GSOC projects, including the possibility of several projects
concurrently if we have multiple students interested in it.

[gsoc_post]: /2012/02/15/gsoc_season_starting.html

Parrot has a lot of introspection ability, but we don't really have the tools
necessary to introspect bytecode. We need some kind of tool that, given a
Sub PMC or a PackfileView PMC or similar will be able to provide a
disassembled representation of the actual opcodes. Here's a basic code
example of what I am talking about:

    function foo() { var x = 2; ... }

    function bar() {
        var disassem = new Parrot.PACT.Disassembler();
        using foo;
        var raw = disassem(foo);
        var reg = raw.registers();          // Get register counts
        var lex = raw.lexicals();           // Get info about lexicals
        var constants = raw.constants()     // Referenced constants
        var ops = raw.opcodes();            // Symbolic Opcodes
        say(ops[0]);                        // "set_p_i, $P0, 2", etc
    }

These are just some random ideas and not all of them are necessary to
implement. The most important part, in my mind, is getting a list of symbolic
Opcode PMCs. Each Opcode PMC would have this general form:

    class Parrot.PACT.Opcode {
        var opname;         // The name or short name of the op
        var opnumber;       // The number of the opcode
        var oplib;          // The oplib which owns it
        var args;           // Array of Arguments
                            // Either Register or Constant
        ...
    }

I prefix the disassembler classname with the namespace "Parrot.PACT" because
eventually this should be an integral component of the PACT library. When we
use PACT to assemble packfiles (and, ultimately, bytecode files) we'll be
constructing a list of these Opcode PMCs and then using a serializer to write
them down to raw bytecode.

                      Serializer
    Array of Opcodes ------------> Packfile Bytecode Segment

              Deserializer
    Bytecode --------------> Array of Opcodes

An excellent proof of concept system would combine these two mechanisms
together into a faithful round-trim assembly/disassembly mechanism. In fact,
there are multiple little potential projects here that can be arranged and
ordered/prioritized to create a summer-long project or many:

1) Create a tool to disassemble raw bytecode into Opcode PMCs, and create a
   disassembler program to interact with the user and print the disassembly
   listings to file/console.
2) Create a tool for round-trip disassembly and assembly. Write the
   disassembly type, then write a tool that does the reverse operation (take
   a list of Opcodes and write a valid Packfile or bytecode segment).
3) Create the tool to disassemble raw bytecode, then write a utility layer to
   construct a control flow graph from those Opcodes. This layer could be used
   in turn to create things like code complexity analyzers, or even
   simple decompilers (for the very ambitious student).
4) Write a tool to take a stream of Opcode PMCs and other related data (tables
   of constant values, annotations, debugging symbols, etc) and write them
   into a valid and executable packfile. This would be the base layer of the
   PACT assembly engine, and would be used to help build compilers and other
   tools.
5) Construct anything else PACT-related (AST and manipulators, CFG/DAG and
   friends, PIR->Opcode assembler, etc). There is lots of fertile ground here
   for projects (and we have a lot of ideas and designs already put together
   for these things).

There are lots of ideas here and I've still only scratched the surface. My
goal with this post is to show how fertile this ground is, how much available
work there is to be had and how many new features we desperately need.

Here's a basic flow graph of things I'm envisioning as eventual parts of PACT
or its close cousins. This will show the kinds of components that PACT may
either eventually contain or serve as the common substrate for:

                                          PIR and PASM Code
                                                  |
                Optimizers         Analyzers <-+  |  +-> Debugger/Live
                ^ |    ^ |           ^         |  |  |            Interpreter
                | V    | V           |         |  V  |
    HLL code -> AST -> Control Flow Graph -> Opcode stream -> Packfile
                 ^       |           ^         |  ^  ^           |
                 |       |           |         |  |  |           |
                HLL <----+           +---------+  |  +-----------+
           (Decompiled)                           |              |
                                                  |              |
                                                 PIR Code <------+
                                              (Disassembly)

One day when I have more time I may try to put this together into a real image
of some variety. ASCII graphics were good enough for our digital ancestors and
they will suffice here for a first draft. As you can see this graph contains
several components, any one of which or any small subsection might make for
an interesting and extremely rewarding project over the summer. This also
ignores the inherent complexity and layered architecture possible in things
like the AST transformations and optimizations, register allocation, etc. My
point is that even the blocks on the graph above can be further decomposed
into a variety of smaller but still interesting projects. If any of this
stuff looks interesting to you, please get in touch ASAP so we can start
talking and planning. Obviously this is more work than one person will do in
one summer, so we want to make sure we are coordinating between all interested
parties.

I think that if we start on the left side of this chart and implement the
routines for reading from and writing to packfiles first, we can start building
layers of additional functionality on top of them. This gives us an ability to
break such a big system up into managable parts, to complete some of those parts
in small summer-sized chunks, and to be able to use intermediate implementations
to solve real problems while we wait for the rest of the system to grow and
mature.

If we had multiple students interested in working on PACT in one capacity or
another it would be an awesome way to maximize developer resources and help
push forward the idea of code reusability. I'm really excited about this whole
area and would love it if some students were interested in it too.
