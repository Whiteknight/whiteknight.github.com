---
layout: post
categories: [Parrot, Winxed]
title: Compile Winxed With Winxed
---

You can compile Winxed code with Winxed itself! What's that you say? The Winxed
compiler is bootstrapped and self-hosted, and is written in Winxed and already
compiles winxed? Well, that's all true. Sort of. However there is one small
caveat: The Winxed driver program historically has not been able to perform the
last step of compilation. The driver compiles winxed code down to PIR, but then
uses the `spawnw` opcode to invoke an instance of Parrot to compile the PIR down
to PBC.

I'm pleased to say that this last step is no longer necessary. At least, not in
Winxed master (which has not yet been snapshotted into Parrot core).

Here's a small toy compiler driver that uses Parrot's PackfileView PMC to
compile a `.winxed` file down into `.pbc` without spawning any child processes:

    function get_winxed_compiler(string pbc_name = "winxedst3.pbc")
    {
        var wx_pbc = load_packfile(pbc_name);
        for (var load_sub in wx_pbc.subs_by_tag("load"))
            load_sub();
        return compreg("winxed");
    }

    function main[main](var args)
    {
        var wx_compreg = get_winxed_compiler();
        string winxedcc_name = args.shift();
        string infile_name = args.shift();
        string outfile_name = args.shift();

        string code = (new 'FileHandle').readall(infile_name);
        var pf = wx_compreg.compile(code);
        pf.write_to_file(outfile_name);
    }

That's less than 20 lines of Winxed code to get the Winxed compiler object
loaded, to compile the code and to output the PBC to file. We can make this
better, of course, by being more flexible in the handling of arguments and
printing out basic help and error messages and all that stuff. Eventually we are
going to update the winxed executable itself to use this trick instead of
spawning the child process. This should, I hope, have a noticably beneficial
effect on compiling with Winxed from the commandline. For large, long builds
like Rosella has, any speed improvements are appreciated.

One particularly interesting tidbit to notice is the very first line: A new
syntax for handling optional parameters. I put a patch for that feature together
last week and NotFound decided he could do the same thing but better than I
did. So, the latest Winxed compiler (again, not yet snapshotted into Parrot
Core) supports this syntax for providing default values to optional arguments. I
hope that this feature is included with the 4.1 release next week.  Here are
some examples of the new feature in action:

    // This...
    function foo(var bar [optional], int has_bar [opt_flag])
    {
        if (!has_bar)
            bar = default_bar_value();
        ...
    }

    // ...is the same as this
    function foo(var bar = default_bar_value())
    {
        ...
    }

The initializer can be any expression value, including expressions involving
previous arguments:

    function foo (var bar, var baz = bar.some_method(bar))

This new syntax probably has a few kinks to work out still, but it's a very
cool and very appreciated addition. I'm hoping to use this new syntax to
clean up a lot of code in Rosella and hopefully in Jaesop too.
