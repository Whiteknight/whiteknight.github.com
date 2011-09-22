---
layout: post
categories: [Parrot, Rosella, CommandLine]
title: Rosella CommandLine
---

I've been working on a new library for Rosella called CommandLine. The
CommandLine library is a response to some of the hassles I have experienced
lately in dealing with command-line argument handling and other issues.

Command line arguments are tricky things. This is partially because there is
no one, single, authoritative standard for how arguments should be specified
or what kinds of forms they may take. There are some pretty common patterns
that are followed, but they are not by any means universal. Parrot provides a
library called Getopt::Obj, which can be used to parse out some of the most
common forms of arguments. Getopt::Obj takes an array of strings that specify
argument names with forms and modifiers, and uses that list to extract
meaningful values from the raw commandline arguments. This library is very
good at what it does, but it's my opinion that it really doesn't do enough.
Plus, since it's implemented as a large wall of PIR code, I don't find it
easy enough or pleasant enough to try to hack in new functionality.

In the new CommandLine library, program details are encapsulated into a
Program object. Program objects have ProgramModes which are used to selected
which behaviors the Program implements. For instance calling a utility program
with the `--help` object should bypass most normal program logic and output
a help message. Calling the `valgrind` utility with the modifier
`--tool=callgrind` changes the behavior pretty significantly. Calling `git
push` has radically different behavior (and expectations about the number and
types of arguments) from `git commit` or `git merge`, even though they all
appear to be calling the same `git` program. These things are called modes.

With CommandLine, we can use these kinds of flag arguments to dispatch to
different entry point functions instead of having to get a list of arguments
and perform the dispatch logic yourself. Here's a short example of the library
in use:

    function main[main](var args) {
        var rosella = load_packfile("rosella/core.pbc");
        var(Rosella.initialize_rosella)("commandline");

        var p = new Rosella.CommandLine.Program(args.shift());
        p.add_mode("merge").set_flag("merge").set_function(merge_main);
        p.add_mode("commit").set_flag("commit").set_function(commit_main);
        ...
        p.run(args);
    }

In the example above we set out two modes, "merge" and "commit" which dispatch
to functions `merge_main` and `commit_main` respectively. If this program is
saved as `mygit.winxed`, you could invoke the two by calling either:

    winxed mygit.winxed merge
    winxed mygit.winxed commit

The first will see the "`merge`" flag and will dispatch to the `merge_main`
function. The second will see the "`commit`" flag and will dispatch to the
`commit_main` function automatically. What if neither is passed? We can set
an error handler to execute when we don't get any valid parameter
combinations:

    p.on_error(show_usage_and_exit)

That line of code will dispatch to the `show_usage_and_exit` function if no
valid modes are found, or if there is a problem parsing arguments.

Now, we all know that we want to have some arguments for these two. We can
pass a hash of named arguments to each mode. These can be required or
optional. We can also provide lists of positional arguments to be expected:

    p.add_mode("merge").set_flag("merge").set_function(merge_main)
        .require_positional("remote", 1)
        .require_positional("ref", 1);
    p.add_mode("commit").set_flag("commit").set_function(commit_main)
        .require_args({ "-m" : "s" })
        .optional_args({ "-a" : "f" });

The arguments to the first mode of the program would look something like this:

    winxed mygit.winxed merge origin master

The second would look more like one of these:

    winxed mygit.winxed commit -a -m "this is a message"
    winxed mygit.winxed commit -m "This is a message"

We can also have a default mode that we fall back to if no other modes can
be matched:

    p.default_mode().set_function(show_usage_and_exit);

So that's how you set up CommandLine argument parsing. The dispatched
functions receive a CommandLine.Arguments object which holds the parsed
information by name:

    function commit_main(var args) {
        int has_dash_a = args["-a"];
        string message = args["-m"];
        string program_name = args.program_name();
        ...
    }

    function merge_main(var args) {
        string remote = args["remote"];
        string ref = args["ref"];
        ...
    }

I recognize that this approach is a heavier-weight solution than what
Getopt::Obj provides. I offer no argument there. What I do like about this
approach however is that it automates many common tasks. We can automatically
dispatch based on parameter combinations, we can automatically detect and
handle errors, we can parse arguments differently depending on what mode we
end up in, and we can specify arguments which are expected to be positional or
named, each required or optional.

Another feature that I've been enjoying is the C-like ability to return an
integer from your main function and have that become the exit code of the
program. For instance, if I have this:

    function show_usage_and_exit(var args) {
        say("You done got the args mixed up!");
        return 1;
    }

The program will exit with an exit code of 1. It's a little C-ism that I've
always wished Parrot supported. Of course, the return value is completely
optional, for those of you who want to keep as much distance between
yourselves and C semantics as possible.

The CommandLine library is still in alpha development, and I have a lot more
features to add. But, I am pretty happy with the way it's progressing right
now, and have already been able to use it to good effect to help clean up some
of the utility programs in Rosella. One thing I do want to do soon is add the
ability to check arguments against matcher functions ("this argument should
look like a number, this a word, and this like an IP address", etc). The more
error checking we can do in the library, the cleaner user code can be and the
less effort has to get put in to argument processing and verification by the
user. Of course, like all Rosella libraries, if the default behaviors don't
fit your individual needs most parts of it should be able to be subclassed
or replaced. I hope that the defaults will be usable for the majority of uses,
however.
