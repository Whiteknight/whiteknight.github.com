---
layout: post
categories: [Parrot, C#, ParrotSharp, Embedding]
title: Introducing Parrot#
---

A few days ago I started a new project that I am provisionally calling
*Parrot#* ("Parrot-Sharp"). This new project provides bindings for Parrot
in C# or other .NET code using Parrot's new embedding API.

A while back I showed an [example on this blog][parrotincsharp] of a very
short toy program, written in C#, which embedded Parrot and printed out a
short "hello world" message. I knew, after having done that example, that I
would be able to get this new project working without too much trouble. The
big saving grace here is that the API works almost exclusively on PMC, STRING,
and simple types, which makes wrapping the function calls very easy. I wrap
the low-level pointers up in custom C# proxy types that include calls to the
native functions in libparrot.

[parrotincsharp]: /2010/04/10/parrot_in_c.html

Tonight, I have an example program that runs. Here's the C# code of the
test executable:

{% highlight csharp %}

using System;

namespace ParrotTest
{
    class MainClass
    {
        public static void Main (string[] args)
        {
            string exename = AppDomain.CurrentDomain.FriendlyName;
            if (args.Length <= 0) {
                Console.WriteLine("No PBC file specified");
                return;
            }
            string pbcfile = args[0];
            string[] pbcargs = new string[args.Length - 1];
            for (int i = 1; i < args.Length; i++)
                pbcargs[i - 1] = args[i];
            Parrot.Parrot parrot = new Parrot.Parrot(exename);
            Parrot.Parrot_PMC pbc = parrot.LoadBytecodeFile(pbcfile);
            Parrot.Parrot_PMC mainargs = parrot.PmcNull;
            parrot.RunBytecode(pbc, mainargs);
        }
    }
}

{% endhighlight %}

It's a simple wrapper program that runs a PBC file. I write a simple PIR file:

    $> cat test.pir
    .sub main :main
        say "Hello from a PIR file!"
    .end

Now, I compile it:

    $> parrot -o test.pbc test.pir

...And run it with my new Program:

    $> .ParrotSharp.exe test.pbc
    Hello from a PIR file!

I actually have a lot more functionality written than what is exercised here,
but I'm having a few problems with MonoDevelop and it's not finding some of
the classes and methods I have written. Once I get some of these thing working
It will be a lot more functional.

I'll write more about this project as it matures, but it is currently
functional enough to execute Parrot bytecode, and I feel like that is a
milestone worth reporting. I also have exceptions working more or less
properly (I've even used my implementation of exceptions here to find a bug
in the embed_api2 branch), and a few other things.

In order to completely replace the Parrot executable I would have to
implement wrappers for IMCC, and frankly I just don't want to do that. For now
the ParrotSharp library is going to provide all other functionality for
working with PMCs, STRINGs, and bytecode. Eventually I'm going to put together
some unit tests too.

Parrot's new embedding API has it's first consumer, I'm excited to see what
other kinds of things people can do with it.
