---
layout: post
categories: [Parrot, Jaesop]
title: Jaesop Modules
---

I've received a few emails from prospective GSOC students interested in doing
Jaesop-related work this summer for GSOC. So, to make sure it was a platform
that's fit to be worked on, I fired it up and ran through the tests. Everything
passed, which is pretty awesome, except there weren't a whole heck of a lot
of tests to begin with. Saying that 100% of tests covering about 10% of the code
passed isn't saying much with any kind of certainty.

Having a student work on Jaesop, if such a proposal is submitted and accepted,
would be quite a boon for that project. In summers past we've had students
working on compiler-related projects, usually starting with nothing or almost
nothing. For instance last year Rohit was working on a JavaScript compiler
starting from the ground up and didn't have a huge amount of luck. Lucian was
working on a Python compiler project, disregarding much of the older Pynie
project work that had been done. He did better, but would have been able to
achieve a lot more if he were starting from a stronger foundation (especially,
an improved Python-ready object model). Asking another student to work on a
new compiler project this year would not be a great thing for us to do,
especially knowing that some of the fundamental issues (i.e. object model) are
still not resolved at the lowest level.

Jaesop is slightly different because a student would be starting with a working
foundation. It's not perfect by any stretch, but it is something. It is a
working piece of code with some of the complexities of the JavaScript object
model and library logic already sorted out. To make it an even better platform
for launching a summer project, I decided that a few more pieces needed to be
added. I feel like it's the difference between a student who can hit the ground
running and one who has to crawl around a few big rocks first.

First thing, I added `require()` and `exports`. This way we can do the Common.js
analog of loading bytecode files. Here's a small example that I wrote out,
called `sys.js`:

    exports.puts = function(s) {
        WX->say(s);
    };

And here is how we would call that from our program:

    var sys = require("sys");
    sys.puts("Hello world!");

Doing this kind of stuff is the kind of unnecessary complexity that a student
shouldn't have to worry about. It's tangential to any problems a student would
be solving and so having the student waste time on infrastructure like this
would just be taking time away from the actual project.

I've also improved logic relating to protypes. It's not perfect or compliant by
any standards, but it's much much closer to the norm and likely gets us close
enough to parsing and running the kinds of non-trivial programs that are going
to form the basis of a compiler.

    var sys = require("sys");
    function Foo() {
        this.a = "foo a";
    }
    Foo.prototype.b = "foo_b";

    function Bar() {
        this.a = "bar a";
    }
    Bar.prototype.b = "bar b";

    var f = new Foo();
    sys.puts(f.a);
    sys.puts(f.b);

    var b = new Bar();
    sys.puts(b.a);
    sys.puts(b.b);

If I didn't tell you otherwise, you might think this was normal JavaScript
code running on Node.js or some other real JavaScript compiler! This exact code
example is running on my machine right now. Prior to my hacking this morning,
the code above would either have thrown an error or have printed out the same
thing twice because both `Foo` and `Bar` function objects would have shared a
prototype.

Now that I've done this work I feel much better about a Jaesop-based GSOC
project happening this summer. Now the responsibility lies with the students to
submit acceptable proposals and get to work!
