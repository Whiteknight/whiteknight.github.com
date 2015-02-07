---
layout: post
categories: [Parrot, Jaesop]
title: Jaesop Runtime Additions
---

I haven't had a lot of time for hacking recently, but I did have a little bit
of time this weekend. I decided that Jaesop needed a little bit of love, so
I went ahead and provided it.

The first thing I did this morning was to add PCRE bindings to the stage0
runtime. Now, if you build Parrot with PCRE support, you can write this kind
of stuff in JavaScript:

    var regexp = new RegExp("ba{3}", "");
    regexp.test("baaa");    // 1
    regexp.test("caaa");    // 0
    regexp.test("baa");     // 0

    regexp = /ba{3}/;       // Same.

Support isn't perfect yet, but it's enough of a start to get us moving with
other things. Specifically, I don't have modifiers like `g` and `i` working
yet, but once I figure out the way to tell PCRE to do what I want it shouldn't
be too hard to add.

If you don't build Parrot with PCRE support, the `RegExp` object won't be
available, and I think it's going to spit out some ugly warning messages.
Considering this is just an ugly bootstrapping stage and it's not complete
yet, I don't mind making these kinds of things optional.

I also added in the beginnings of a runtime. Now there are some basic objects
like `Process` which you can use to interact with the environment, and
`FileStream` which you can use for input and output. The `process` variable
is always available as a global, and it gives access to the standard streams
and command-line arguments. Here are some examples:

    process.stdout.writeLine("Hello World!");
    process.stdout.writeLine(process.argv[0]);
    var s = process.stdin.readLine();

And so, after several weeks of development, you can finally write a simple
"Hello World" program in Jaesop.

What my hacking today has shown me is that the one thing I am severely lacking
on are the tests. I do have some tests but I don't have nearly enough.
Specifically, I've learned that my test coverage of Arrays is severely
inadequate. I also need to test a few other details which I found out were
horribly broken when I went to play with them today.

Despite some of the setbacks, Jaesop stage 0 compiler is progressing nicely
and it's getting to the point when we can start to do some real work with
it. These few runtime additions, though small, have greatly improved the
situation. There are a few things I still need to do to it, besides the
testing I mentioned above: I need to improve the PCRE bindings because right
now they are very basic. I need to add methods to Object, Array, and String
for usability. I also need to add in a mechanism like node.js' `require`
routine to load modules, and maybe a few other similar code management
details. When that stuff is all done, and when Parrot has proper 6model
support, we can start moving forward on the stage1 compiler. I'm really
starting to get excited about taking that next step.
