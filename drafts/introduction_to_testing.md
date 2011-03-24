In my previous posts about ideas for GSoC projects, I've mentioned the need to
do proper testing and have proper test suites as part of the project
deliverables. Some prospective students asked about it on IRC, and I decided
it would be a good idea to write up a tutorial of sorts about testing: What
it is, why we do it, why it's important, and how to get started with it in
your GSoC project.

As part of this tutorial I am going to be showing most of the examples using
Rosella, because... well, that's just the project that I use for my testing.
This isn't the only tool to be used, but it will help to keep several of my
examples brief. I'll be writing code examples in both Winxed (a low-level
Parrot language which is similar to JavaScript) and in NQP (a low-level
subset language of Perl6). Most students under the Parrot organization are
going to be writing software in multiple languages anyway, so it's good to
just start that mindset here.

## What is Testing?

Testing is the process where we write programs to test our real programs. What
we want to do is start executing real code from our software in bite-sized
chunks, and verify that it does what we think it should do. Testing serves
two primary purposes:

1. Verifies that the software works as intended, including across different
   platforms.
2. Verifies that the software *continues to work* even after we make changes.

In the case of a large complex project like Parrot, unit testing helps to make
sure that your software works as intended even if some of your dependencies
change. As a quick example, if you do indeed use Rosella as part of your
project or your test suite, a test will show if something changed in Parrot
which caused a breakage or if something changed in Rosella which caused a
breakage. Even if you do nothing wrong (and we all make mistakes, that's why
we have debuggers), your software could still break and you want to know about
it as soon as possible.

There are several different concepts in regards to testing. Many of them
overlap. Here's a quick list of terms which you will probably find as you talk
to people and start research and real coding work:

1. *Unit Testing*: You break your software down into the individual components
   (classes, methods, functions, etc), and test each one individually.
2. *Integration Testing*: This can mean different things to different people,
   but typically it involves combining together your units to verify that they
   work well together under real-world kinds of inputs.
3. *Regression Testing*: In Regression Testing, you write tests not to find
   old bugs, but to prevent new ones. Write tests to prove that your software
   works, then if it ever breaks in the future the tests will show it.
4. *Test Driven Development*: (TDD) is a style of software development where
   you write the tests first. Write the test first and make sure they fail.
   Then write and fix the code until the tests pass. Once the tests pass, the
   software is done (and, you're protected from regressions).
5. *Smoke Testing*: The way we use the term in Parrot world is that you submit
   test reports ("smoke reports") to a server somewhere, so everybody can see
   when software is broken. This helps to identify platforms and
   configurations which are broken, and allows everybody to participate in
   the process by submitting and reading reports.
6. *Test Coverage*: The precentage of your code which is executed by your
    test suite. The higher the test coverage, the more certain you can be
    that your software works the way it is expected to work. You can never be
    100% certain about anything (your tests could contain bugs, and you aren't
    going to test all your tests!), but you can get pretty close sometimes.

With the basic concepts out of the way, let's talk about how to test

## How to Test

To test your software you need two components: A set ("suite") of tests to
run, and a test harness with which to run them. Actually, you need a third
piece too: Your actual software to be tested.

By convention, all the source code for our project will be in a folder called
"src", and all your tests will be in a folder called "t". You could name these
whatever you want, really, but this is a nice common convention and I do
recommend you stick with it unless you have some kind of major problem with
it.

The exact way to organize your tests is also up to you. Again, by convention,
each code file or each code class would get a corresponding test file. So if
you have a file `src/stuff.c`, you would have a test file `t/stuff.t`. The
`.t` suffix is another part of the convention.

The idea of 1 test file per 1 source file/class is a good one, but really
doesn't offer the flexibility you will want. For instance, if you are working
on one of the compiler project ideas, you will probably want to unit test the
source, but then also have a variety of tests of language syntax, or tests
of embedding your compiler in another program, or tests for embedding other
programs in your compiler, or whatever other combinations you can think of.

Here's an example folder hierachy for your project:

    /include        # .h header includes, if needed
    /src            # Source code
        /pmc        # custom PMC declarations
        /ops        # custom ops declarations
    /lib            # Runtime library, extensions, addons, etc
    /t
        /src        # tests arranged by source code file
            /pmc
            /ops
        /lib        # tests for the runtime library
        /classes    # Tests for individual classes
        /syntax     # Tests for various syntax, etc
        /benchmarks # Tests which exercise the software and do timings

Again, this is just an example and you can take as much or as little away from
it as you want. People do follow common conventions for a reason, and unless
you have really good reasons for not wanting to do things this way, it's
probably better to just stick with it.

Also by convention, your harness is typically named `t/harness`. I won't get
into the details about how to write a harness here. That's not really an
important exercise. There are several existing tools for writing harnesses and
even some good pre-existing harness code to copy from, so it's not the kind of
thing you will want to be wasting much effort on.

## Testable Code

The basic idea behind a test is that you call some of your code with certain
inputs and get back the things that you expect to get back. Of course, things
are rarely as clean as a function which returns values, doesn't modify the
data references you pass in as parameters, and has no other side-effects.

This in mind, we need to write software in such a way that it is *testable*.
This means we separate out logic into bite-sized pieces which we can analyze
individually. It also means that we should subscribe to ideas like the
"single responsibility principle". If each function only does one thing, we
only need to test that one thing. If your function does a million things, we
need to write millions of tests for it.

Let's say I have two functions, each of which does 5 different possible
things. One function calls the other. In this system, there are 25 possible
combinations, all of which need to be tested. Now, let's break all of those
down, and we have 10 functions which each only do one thing. Now we only need
one test. Plus, our software is cleaner and our tests are more clear. This is
a good thing.

Here's an example of a function which does a few different things (I'm writing
this example in Winxed):

    function DoStuff(var a, var b) {
        var c = a + b;
        a.doSomethingElse(c, b);
        say("I just did stuff with a: " + a);
        return c;
    }

So this function does three things: It creates the new variable `c = a + b`,
it calls the `a.doSomethingElse(c, b)` method to update the variable `a`, and
it outputs some stuff to the console to tell the user what happened. This is
a trivial example of course, but we can still rewrite this to make it more
testable:

    function DoStuff(var a, var b) {
        var c = Combine(a, b);
        a.doSomething(c, b);
        TellUserAboutIt(a);
        return c;
    }

Now, we have four functions to test: `DoStuff`, `Combine`, `a.doSomething`
and `TellUserAboutIt`. This makes our testing much easier. If we completely
unit-test the functions `Combine`, `a.doSomething` and `TellUserAboutIt`, we
can be pretty certain that `DoStuff` is going to work correctly, even without
writing any tests for `DoStuff` directly. We will still want to write tests
for it, but we have all the internal stuff covered.

But wait! We're not done. How do we test `TellUserAboutIt` in an automated
way? We want to capture the text that it is printing out, to make sure the
user is being told the correct things. So, let's update that method to take
a handle to an output object. In the normal case, this will be a handle to the
`stdout` console file, but it doesn't have to be.

    function DoStuff(var outfile, var a, var b) {
        var c = Combine(a, b);
        a.doSomething(c, b);
        TellUserAboutIt(outfile, a);
        return c;
    }

    function Combine(var a, var b) {
        return a + b;
    }

    function TellUserAboutIt(var outfile, var a) {
        outfile.write("I just did stuff with a: " + a);
    }

With this, we now have software which is completely testable. Next I'll show
how we write those tests.

## Writing Tests

So we have our routines above and would like to test them. Let's start by
writing up a basic empty test file with some of the necessary infrastructure.
I'll write this part in NQP:

    INIT { pir::load_bytecode("rosella/test.pbc"); }

    Rosella::Test::test(MyFunctionTests);

    class MyFunctionTests {
        method DoStuff() { }

        method Combine() { }

        method TellUserAboutIt() { }
    }

In Rosella's test system, every method is a test. Test methods take no
parameters and return no values.

In other systems such as `Test::More`, there are different ways to specify
tests. `Test::More` is, in my opinion, a little bit more complicated to use
but it gives you a little bit more control over certain situations. I won't
show it here because of the complexity, but I could show examples of it in
another post if people are interested in reading about it.

Now that we have our basic file skeleton, let's start filling in the details.
First, let's fill out the test for `TellUserAboutIt`:

    method TellUserAboutIt() {
        my $output := pir::new__PS("StringHandle");
        TellUserAboutIt($output, "TEST");
        my $str := ~$output;
        Assert::equal($str, "I just did stuff with a: TEST");
    }

We'll go through this test line-by-line so there are no surprises. In the
first line we create a new variable called `$output` which holds a
StringHandle reference. A StringHandle is like a FileHandle, but instead of
writing to a file or to the console we write to a String.

On line 2 we pass this StringHandle to our `TellUserAboutIt` function, along
with some dummy test value.

In line 3 we get the string that we wrote into a new variable `$str`.

In line 4, we call the function `Assert::equal`. This is the actual
verification part of the test. If the two items are not equal the test fails.
If they are equal, the test passes. In this case, I think the test would
pass, unless I made a stupid typo somewhere.

## Testing For Error Handling

Just for good measure, let's create a new method to test what happens if we
call `TellUserAboutIt` with a number instead of a string argument.

    method TellUserAboutIt_with_a_number_instead() {
        my $output := pir::new__PS("StringHandle");
        TellUserAboutIt($output, 5);
        my $str := ~$output;
        Assert::equal($str, "I just did stuff with a: 5");
    }

Notice how I gave the new method a new name to describe what it does. Also
notice that on line 2 I changed the string "TEST" to the number 5, and on
line 5 I changed the string we expect to get back. When I run this test now,
it fails.

LOLWUT?

The answer is in the definition to the `TellUserAboutIt` function. In Winxed,
using the `+` operator on a number and a string will produce a number result,
not a string. In the test, the variable `$str` contains the value 5, not the
string we expect. So, let's fix the code by casting the value to a string:

    function TellUserAboutIt(var outfile, var a) {
        outfile.write("I just did stuff with a: " + string(a));
    }

Recompile. Re-run the test. Pass. All is good with the universe.

We can even write a new test, to see what happens if we pass in something
stupid for the first argument, instead of a valid handle type. That may be a
little bit unnecessary right now, I know what is going to happen. We're going
to get an exception telling us that the weird object I passed in doesn't have
a method "`.write()`". However, if I add in some custom error handling to my
routine:

    function TellUserAboutIt(var outfile, var a) {
        int is_a_handle = can(outfile, "write");
        if (!is_a_handle)
            return;
        outfile.write("I just did stuff with a: " + a);
    }

Now that I have some custom behavior in place, that's worth testing. Let's
write a basic test that the example now doesn't throw that "method not found"
exception I mentioned earlier:

    method TellUserAboutIt_with_bad_handle() {
        Assert::throws_nothing({
            TellUserAboutIt("I'm a String, not a handle", "TEST");
        });
    }

Notice we're using a different assertion now, one that tells us that the
inside stuff doesn't throw an exception. If we throw an exception, the test
fails, otherwise, it passes. Since the function shouldn't do anything, this
is a fine test.

## Conclusion

This has been just a short demonstration about testing, but it should be
enough to get most students started writing at least basic unit tests. At the
very least, it should help you figure out what kinds of things to include in
your research in the days ahead.

The real benefit of testing is that if you write tests as you go, you can be
sure that the code you are writing actually works, and that you aren't
breaking old things as you write new ones. Plus, if you write tests while you
code, you aren't going to have to write a million of them at the end of your
project, when the deadline is looming.

I definitely recommend you get into the habit of writing a feature, then
writing a test to prove it works. Write a feature, write a test. Write a
feature, write a test. Repeat until you're done. Alternatively, if you're into
the whole TDD thing (and a lot of very good coders love it), you write a test
then write a feature, etc.

Testing is a way for you to say "My code works, and I can prove it". That's
a very powerful and reassuring thing. The last thing you want is to come down
to the deadline of GSoC and not have any idea if your software does what you
want it to do. The Test suite is a required part of your deliverables anyway,
so you may as well use them as a powerful tool to help you as you code.
