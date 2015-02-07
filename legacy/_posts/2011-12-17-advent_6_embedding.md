---
layout: post
categories: [Parrot, Advent2011]
title: Advent 6 - Embedding
---

In late 2010 and early 2011 we spent a good amount of effort building a new
embedding API for Parrot. I would like to say that the new API replaced an
older, inferior API but that's not really the case. We didn't really have an
old embedding API per se. We had a mismash of functions in a file called
`embed.c`, but they hardly represented a consistent API, much less a complete
set of things that an embedder would need. If anything the old embedding API
was the entirety of all publicly exported functions from libparrot combined
with a handful of utility functions that embedders in the past also needed.

In short, it was a mess. By early 2011 we had a much nicer API around to play
with. Now that 2011 is almost over, the new API is considered to be extremely
stable and robust.

Last December I started a project called ParrotSharp which embeds a Parrot
Interpreter into the .NET CLI with C#. I haven't shown that project too much
love in recent months, but as of today it's still building and seems to run
correctly (although my IDE is telling me it can't find NUnit on my system, so
it won't run my tests). That has to tell you something, when code I wrote
months ago with the embedding API still works correctly even though so many
things have changed in Parrot since then.

Parrot's embedding API is a little bit verbose but very easy and straight
forward to use. Also, all API functions return a true/false status value, so
calls can easily be chained together. Here is an example of the embedding API
in action:

    int main(int argc, char** argv) {
        Parrot_PMC interp, bytecodepmc, args;
        Parrot_Init_Args *initargs;
        Parrot_String filename;

        GET_INIT_STRUCT(initargs);

        if (!(
            Parrot_api_make_interpreter(NULL, 0, initargs, &interp) &&
            Parrot_api_set_executable_name(interp, argv[0]) &&
            Parrot_api_pmc_wrap_string_array(interp, argc, argv, &args) &&
            Parrot_api_string_import(interp, "foo.pbc", &filename) &&
            Parrot_api_load_bytecode_file(interp, filename, &bytecodepmc) &&
            Parrot_api_run_bytecode(interp, bytecodepmc, args)
        )) {
            Parrot_String errmsg, backtrace;
            Parrot_Int exit_code, is_error;
            Parrot_PMC exception;

            Parrot_api_get_result(interp, &is_error, &exception, &exit_code, &errmsg);
            if (is_error) {
                Parrot_api_get_exception_backtrace(interp, exception, &backtrace);
                // Print out exception information to the console, or whatever
            }
        }

        Parrot_api_destroy_interpreter(interp);
        exit(exit_code);
     }

This is, essentially, a simple program to execute a bytecode file. However, it
does show some of the basics of the embedding API. Every function returns a
true/false, pass/fail status bit. All data types passed around are properly
wrapped Parrot_PMC or Parrot_String types and it almost never uses any other
raw pointer types. Also since we're using PMC and STRING types, Parrot's GC
manages all the memory for you and you don't need to be freeing things or
cleaning things up (except for the interpreter itself).

This example above only shows a handful of API functions, but there are several
dozen of them in the API and more can be easily added. We have API routines for
performing a variety of actions on Strings and PMCs. We have API routines for
loading, executing and writing bytecode. The API has decent defaults so you can
just get an interpreter up and running quickly if you want, but it also have a
variety of routines for tweaking and configuring the interpreter too. And, like
I said (and I'll say it a million times more if I have to) we can always add new
methods to the embedding API if there is a need.

We have at least the basics of a C# wrapper projects in the wings, and I've been
planning a proper C++ wrapper for a while too but I haven't gotten around to
it yet. That would make an excellent, smallish project for an intrepid newcomer
to work on, especially one that knows C++. I like to think it should be easy to
embed Parrot as a plugin for things like text editors or other pluggable unixy
programs, but I haven't taken the time to really dig into any of them yet. This
might make another great project for an eager new parrot hacker.

