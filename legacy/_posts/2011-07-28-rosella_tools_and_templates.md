---
layout: post
categories: [Parrot, Rosella, Template]
title: New Rosella Tools and Templates
---

A few days ago [I wrote about a new library][template_beta] I have been
working on called Rosella Template. Template is a text templating library
similar in concept to Liquid, and drawing significant inspiration from the
likes of PHP, ASP.NET webforms, and other tools that allow templating textual
data. I won't go into all the details of the library, I talked about it a good
deal in my previous post and will talk about it much more when I get closer
to declaring it "stable".

[template_beta]: /2011/07/06/rosella_template_beta.html

The library is fun to work on all by itself, but it is enabling me to make
some very cool tools. Also, new packfile features in Parrot are allowing me
to make even more tools. Rosella now has a small selection of utility programs
in addition to its collection of libraries, and the number is growing. Today
I'm going to talk about some of the fun new things that I am working on. All
of them are still marked "unstable", but as I play with them more, document
them, and get some feedback they could move up to "stable" status.

`mk_winxed_header.winxed` is a utility program for creating
forward-declaration include files for Winxed. If you're doing a lot of hacking
with Winxed, this tool is a must. It uses the new packfile features in Parrot
to open up a compiled .pbc file, extract records of all the Sub, NameSpace
and Class definitions in it and write out a .winxed include file. The usage
would be something like this:

    winxed mk_winxed_header.winxed foo.pbc > foo.winxed

And, depending on the contents of that .pbc file, the output will look
something like this:

    namespace Foo {
        class Bar;
        class Baz;

        extern function foobar;
        extern function barbaz;
        ...
    }

In your winxed programs, if you are using foo.pbc, you can do this, to make
the "type not found at compile time" warnings go away:

    $include "foo.winxed"

    function main[main]() {
        var bar = new Foo.Bar();
        var x = Foo.foobar();
    }

This routine only uses the parrot functions and currently doesn't make much
use of any Rosella functionality, because I originally wrote it up as a
general demonstration of the new Parrot features.

`test_template.winxed` is a simple tool for creating test-related files from
templates. Rosella ships with a few pre-made templates, which are used by this
script. `test_template.winxed` can be used to create either a test file or a
working harness file with nothing but a few commandline options. If you want
a test harness, you can do something like this:

    winxed test_template.winxed harness nqp > t/harness

And like magic you have a basic, working test harness, written in NQP. You can
use the argument "winxed" instead to get a harness written in winxed. The two
are functionally equivalent.

If you have written and compiled a new class, you can generate tests for it
very easily:

    winxed test_template.winxed test winxed foo.pbc Foo.Bar > t/foo/Bar.t

The first argument says that we're making a test, not a harness. The second
argument says it should be written in Winxed (can also be "nqp"). The third
argument is the name of the .pbc file to read, and the final argument is the
fully-qualified name of the class to look up. What you get as the output
is a file very similar to this:

    // Automatically generated test for Class Rosella.Template.Node.Data
    class Test_Foo_Bar
    {
        function test_sanity()
        {
            self.assert.is_true(1);
        }

        function test_new() {
            var obj = new Foo.Bar();
            self.assert.not_null(obj);
            self.assert.instance_of(obj, class Foo.Bar);
        }

        function method_A() {
            self.status.verify("Test Foo.Bar.method_A()");
            var obj = new Foo.Bar();

            var arg_0 = null;
            var arg_1 = null;
            var arg_2 = null;
            var result = obj.method_A(arg_0, arg_1, arg_2);
        }

        ...
    }

    function main[main]()
    {
        load_bytecode("rosella/test.pbc");
        load_bytecode("foo.pbc");
        using Rosella.Test.test;
        test(class Test_Foo_Bar);
    }

I've only shown one tested method above for brevity, but it will automatically
generate a stub test method for every method in your class. Also, notice that
it reads the number of expected parameters from each Sub and automatically
creates variables for each.
There are some kinks to work out here, but
this can certainly save a lot of typing!

If you only need to generate a harness, or only need to generate a single test
file, then `test_template.winxed` is the utility for you. If you are starting
from scratch and need to create an entire test *suite*, you want something
with a bit more capability. In those cases, use `test_all_lib.winxed`.
`test_all_lib.winxed` reads in from a .pbc file again, and outputs a test
suite for it. It outputs a test file for every single namespace with visible
Subs, and every single class it can find. Every one of the files will look
similar to the above listing. I won't try to copy+paste the entire output of
the script, but here is a quick example of me running it on the Rosella
MockObject library:

    <rosella git:master> winxed --nowarn test_all_lib.winxed rosella/mockobject.pbc motest
    START NameSpace 'Rosella.MockObject'
        Built Subs test file motest/Rosella.MockObject.t
        Built Class test file motest/Rosella.MockObject/Controller.t
        Built Class test file motest/Rosella.MockObject/Expectation.t
        Built Class test file motest/Rosella.MockObject/Factory.t
    END NameSpace 'Rosella.MockObject'
    START NameSpace 'Rosella.MockObject.Controller'
        Built Class test file motest/Rosella.MockObject.Controller/Ordered.t
    END NameSpace 'Rosella.MockObject.Controller'
    START NameSpace 'Rosella.MockObject.Expectation'
        Built Class test file motest/Rosella.MockObject.Expectation/Get.t
        Built Class test file motest/Rosella.MockObject.Expectation/Set.t
        Built Class test file motest/Rosella.MockObject.Expectation/Method.t
        Built Class test file motest/Rosella.MockObject.Expectation/Invoke.t
        Built Class test file motest/Rosella.MockObject.Expectation/Will.t
        Built Class test file motest/Rosella.MockObject.Expectation/With.t
    END NameSpace 'Rosella.MockObject.Expectation'
    START NameSpace 'Rosella.MockObject.Expectation.Will'
        Built Class test file motest/Rosella.MockObject.Expectation.Will/Return.t
        Built Class test file motest/Rosella.MockObject.Expectation.Will/Throw.t
        Built Class test file motest/Rosella.MockObject.Expectation.Will/Do.t
    END NameSpace 'Rosella.MockObject.Expectation.Will'
    START NameSpace 'Rosella.MockObject.Expectation.With'
        Built Class test file motest/Rosella.MockObject.Expectation.With/Args.t
        Built Class test file motest/Rosella.MockObject.Expectation.With/Any.t
        Built Class test file motest/Rosella.MockObject.Expectation.With/None.t
    END NameSpace 'Rosella.MockObject.Expectation.With'

For those of you who aren't keeping count, this utility generated seventeen
separate, complete, runnable test files. When I think back to all the time I
could have saved having test file skeletons automatically generated instead of
having to write them all out by hand...

At the moment the `test_all_lib.winxed` tool is very early in development and
doesn't output NQP files yet. I need to update all the necessary templates to
support NQP, and add in a new argument to the commandline for this. Notice
also that Winxed and NQP are not the only two options for this or the other
utilities I've discussed here. Writing up templates for other languages is
usually a pretty quick and easy thing to do.
