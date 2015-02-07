---
layout: post
categories: [Parrot, Embedding]
title: Embedding API Home Stretch
---

As of yesterday, with substantial help from contributor [bluescreen][], the
`embed_api` branch completely builds. Not all tests are passing yet, but we're
passed the first major hurdle.

[bluescreen]: https://github.com/bluescreen10

Despite reaching this milestone, there are some big questions left to answer.
Many of the remaining issues are intimately involved with the remaining
failing tests.

Let me show you an extremely simple test that is failing, explain why it is
failing, and talk about ways to make it not fail. First, the code sample:

    .sub main :main
        exit 2
    .end

The test checks that the exit code of the Parrot executable after running this
short example is actually 2. At the moment it is not and the test fails.

Let's try to get an understanding about *why* this fails and why it's such
a pain to fix. To do that, we first need to understand exactly what it means
to exit a Parrot program. How do we exit a program?

There are a few ways that you can exit a program. Here are some examples. The
first is a very common example:

    .sub main :main
        ... PROGRAM LOGIC HERE ...
    .end

IMCC, the PIR compiler front-end to Parrot will automatically add an `exit 0`
op to the end of a `:main` function if it doesn't already have one. So, the
above example is functionally equivalent to this:

    .sub main :main
        ... YOUR CODE HERE ...
        exit 0
    .end

Returning a 0 from a main function with the Parrot executable is the same
semantically as returning a 0 from `int main()` in C. The operating system
typically interprets a 0 return value as a "success". Anything else is an
error condition. In BASH, for instance, the following chain checks the return
value and only executes the next item in the list when the previous one
returns 0.

    parrot foo.pbc && parrot bar.pbc && parrot baz.pbc

If any step in the chain returns non-zero, execution of that compound
statement ends without executing the rest of the statements.

A non-zero exit doesn't necessarily mean something bad happened. For instance,
typing

    parrot -h

at the command line will show the help and exit with a return code of 1. It's
not really an error, but at the same time it isn't a normal result. So, we
can't always treat this condition like an error, sometimes it's perfectly
normal.

Keeping that in mind, many programs run by the Parrot executable may return
non-zero values to indicate error conditions. This helps Parrot to work with
the operating system in executing chained statements. Since the Parrot
executable is basically a thin wrapper around libparrot and a single
interpreter instance, whatever return value comes out of the PIR code is
shuffled directly to the return value of the Parrot executable and passed to
the OS. Return values from programs can typically be in the range of 0-255, so
values 1-255 all indicate possible error conditions. Here is a snippet showing
one example of an error code in that range:

    .sub main :main
        ... YOUR CODE HERE ...
        exit 1  # Tell the OS we've had an error!
    .end

In PIR the previous two examples are very similar syntactically, but the
result according to the operating system is very different. There is also a
third option for how to exit a program:

    .sub main :main
        ... YOUR CODE HERE ...
        throw "This is an Exception!"
    .end

Here we have an exception we are throwing but there is no defined handler to
catch it. In these situations libparrot currently panics and calls the
`exit()` function from the C standard library. This is very bad, and extremely
unfriendly to most embedding applications. In the `embed_api` branch,
libparrot handles this much more gracefully by jumping directly to an error
handler and returning a [non-zero failure status][funcreturns] from the
current API function. The embedding application can then continue it's own
operations, possibly even attempting to retry the operation or start a new
operation in the interpreter.

[funcreturns]: /2010/11/06/embedding_api.html

So where is the problem? The problem is that the `exit` opcode (and the
related `die` opcode) don't just exit: They *throw a special type of exit
exception*. This is great for programmers: They can create an exit exception
handler and catch attempts to prematurely exit the program, installing cleanup
tasks or even aborting the exit attempt entirely. This is bad for the API
because we have a few exit conditions that we need to differentiate between
and communicate that information to the embedding application.

Short recap. Here are the three possible error conditions:

1. A normal (or implied) `exit 0`. This throws an `EXCEPT_exit`.
2. An `exit` with a non-zero value to communicate an error or a special
   situation.
3. An unhandled exception of a type besides `EXCEPT_exit`.

In reality all of these are unhandled exceptions, but the first two are
special cases where things haven't gone horribly wrong and the programmer has
lost control. Underneith they are the same, but on the surface they are worth
distinguishing. Now, here are the ways the current Parrot executable handles
all these things:

1. Exit the program, return 0 to the operating system
2. Exit the program, return the exit code to the operating system
3. Print an error message (and maybe a backtrace or other diagnostics) to
   `stderr` and exit the program. Return the exit code from the exception
   object to the operating system.

When any API call returns, we have several things to communicate: Whether the
partcular API call succeeded or failed, what the exit code was, and if there
was a failure we need to return an error message to print to `stderr`.

Remember, a non-zero exit code does not necessarily mean "error". Also, since
an unhandled exception object contains it's own error code, it's possible
(though a little rude) for a program to have an unhandled exception *and still
return 0*.

Obviously we need two numbers for the status *and* the exit code. Right now,
the new API is only communicating the former. We are returning an error
message through the `Parrot_api_get_last_error` function, but I'm thinking
that's not what we want in the end. Instead, I'm thinking we want something
like `Parrot_api_get_result` that will return the exit code, an error status
flag, and the text of an error message if any. Here's an example of what will
probably be in the Parrot executable to deal with these situations:

{% highlight c %}

if (!Parrot_api_do_some_stuff(interp, ...)) {
    if (!Parrot_api_get_result(interp, &is_error, &exit_code, &err_message)) {
        fprintf(stderr, "So broken we can't even get an error message");
        exit(EXIT_FAILURE);
    }
    if (is_error) {
        const char * szerr = NULL;
        Parrot_api_string_export_ascii(interp, err_message, &szerr);
        fprintf(stderr, "%s\n", szerr);
        Parrot_api_string_free_exported_ascii(interp, szerr);
    }
    exit(exit_code);
}

{% endhighlight %}

Other embedding applications will probably have different behavior, which is
fine: Parrot gives you plenty of options!

Astute observers will notice that the API function `Parrot_api_run_bytecode`
will always return non-zero because there's no way to get out of executing the
bytecode without either an `exit` or an unhandled exception. This is part of
the reason for the elaborate status-checking routine I show above. Doing
things this way means that in the future there is a door open for another
option, where we leave the runloop without some kind of unhandled exception of
one form or another. This could even include a situation where we can stop
execution in the middle of a program and continue running it later. All the
possibilities for this kind of interface are outside the scope of this post
and the current `embed_api` branch, so I won't talk about it now.
