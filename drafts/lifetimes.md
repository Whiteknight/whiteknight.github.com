dukeleto sent out a link to a very interesting blog post about
[Lifetimes inthe Rust programming language](http://pcwalton.github.com/blog/2012/04/23/why-lifetimes/).
Actually, the blog was a background piece about some of the information that
would lead to lifetimes, but a careful reading suggests where they are headed
in subsequent posts.

I think what the author is driving at is something like the `using` structure
in C#. That structure looks something like this:

    using (IDisposable foo = new Foo()) {
        do_something_with(foo);
    }

The variable `foo` is created in the `using` statement, and at the end of the
block is is guaranteed to be disposed of and to have its resources freed. This
is basically syntactical sugar for something like this:

    {
        IDisposable foo = new Foo()
        try {
            do_something_with(foo);
        } catch (Exception e) {
            throw e;
        } finally {
            foo.Dispose();
        }
    }

The thing about the `using` block in C# is that it's guaranteed to free the
object no matter what, whether we do a return from inside that block, or we
throw an exception and try to jump to somewhere else, or whatever. No matter
what, when control flow leaves that block, we dispose of `foo` and its
resources.

This is very similar to a long-standing ticket we've had in Parrot world about
cleanup callbacks: The ability to execute some kind of callback or handler on
subroutine exit. The idea being that we would like to be able to automatically
and reliably do things like close open sockets and file handles when we leave
the current scope. We've never been able to find a good way to do this, and a
large part of the reason why not is because Parrot uses CPS, which when
leveraged in an idiomatic way creates a number of tricks that create many
headaches.

Let's say we had something like the `using` statement above in Winxed. That
statement would, through whatever mechanism, force the variable in question to
be disposed of. Maybe we add in a `.dispose` method as a kind of standard role,
or maybe we null out the reference and hope GC picks it up, or maybe we even
force GC to collect it immediately (and remaining references to it be damned!).
I suspect it would be best to use the `.dispose` method and null out the
reference so GC can try to collect it, but that's a topic for a different
discussion.

Let's look at a few examples where trouble brews. First, an obvious one:

    function foo() {
        using (var bar = new Bar()) {
            return function() { return bar.data(); };
        }
    }

This is a basic closure situation. It should probably be pretty clear that
the variable `bar` will be disposed of when function `foo` returns, but before
the anonymous returned function ever gets the chance to execute. In that
function, we would probably end up with `bar` being null. This is the kind of
thing that, if we could detect it reliably, should be a compile-time error.

Of course, if we attach the cleanup handler to the `foo` context, we can force
GC to dispose of `bar` when the `foo` context is collected by GC. This might not
happen immediately when `foo` exits, of course, but it would happen eventually.
If we attach the cleanup handler to the context and treat the `using` directive
as a kind of closure, we could generate code something like this JavaScriptish
mess:

    function foo() {
        return (function (var bar) {
            var ctx = get_context();
            ctx.attach_cleanup_handler(function() { bar.dispose(); });
            {
                return function() { return bar.data(); };
            }
        })(new Bar());
    }

We use the closure for the `using` block because the `foo` context might create
other closures, and its context would be artificially kept alive by other
things and we would want the `bar` variable disposed as soon as it is not
available any more, which might be sooner than the parent context is destroyed.

Let's also ignore for a minute the fact that the cleanup handler is itself a
closure in this example, which would reference the outer closure, and would
create all sorts of problems, especially if the GC collects the handler closure
before it collects the context and attempts to execute the handler. And with
all that, if the GC has collected all these things, it has probably also already
collected `bar` or added it to its list of things to be collected shortly
thereafter.

Let's look at an example now using a `setjmp()` function which uses
continuations to perform behaviors similar to the C-library namesake. The first
time you call it, it returns a continuation. When you invoke that continuation,
`setjmp` returns again, this time with a null value. This example is a little
bit longer.

    function main() {
        var jmpbuf = setjmp();
        if (jmpbuf == null) {
            // We've jumped and are back!
            say("Done");
            return;
        } else {
            using (var bar = new Bar()) {
                do_something_with(bar, jmpbuf);
            }
        }
    }

    function do_something_with(var bar, var jmpbuf) {
        really_do_something_with(bar);
        jmpbuf();
        // Notice, no return here!
    }

In this example, control flow has left the `do_something_with` function, but
we haven't hit a return or thrown an exception. That context is left and we
never return to it, and there's no way for the compiler to know that the invoke
of `jmpbuf` isn't just a normal Sub call, and that control flow isn't coming
back, ever. In this case, the only real way we can determine that the context
has exited and that the bar value needs to be collected is by having the GC
detect that the context is dead and attempt to collect it. Of course, if the
context is kept alive by some closure somewhere, that might never happen!
Remember that a Context keeps track of two pointers: First is the caller and
second is the outer. The outer is used by a closure to keep track of where its
lexical variables live. If `really_do_something_with` creates and stores a
closure, it's outer points to the `do_something_with` context, whose caller
points to the context created by the `using` block, which is forced to stay
alive as long as the inner closure stays alive.

And we obviously can't just call cleanup handlers when the using block calls
`return`, because as the example above shows, we never return.

But here's an idea. When Parrot invokes a continuation we rewind the environment
and there's an opportunity for us to try and iterate over the contexts between
the jump location and the target destination. We could use that opportunity to
find and execute cleanup handlers, maybe. However, let's make our example above
a little bit more complicated:

    function main() {
        var jmpbuf = setjmp();
        if (jmpbuf.is_jumped) {
            // We've jumped and are back!
            say("HERE");
            var returnjmp = jumpbuf.data;
            returnjmp();
        } else {
            using (var bar = new Bar()) {
                do_something_with(bar, jmpbuf);
            }
        }
    }

    function do_something_with(var bar, var jmpbuf) {
        really_do_something_with(bar);
        var returnjmp = setjmp();
        if (returnjmp != null) {
            jmpbuf.data = returnjmp;
            jmpbuf();
        } else {
            // We've magically returned
            do_something_else_with(bar);
            return;
        }
    }

Now things are all kinds of crazy! We do jump out of the `do_something_with`
context, but we take a continuation and jump right back. That means we can't
dispose of `bar` when we jump out of a context, because we might be able to jump
right back to it. This is a very similar mechanism to resumable exceptions,
which Rakudo does make use of, so this example isn't entirely contrived. Plus,
if Parrot really does intend to use CPS as a strength, and attracts other
languages which rely on CPS more heavily than does Rakudo, these kinds of
mechanisms will become commonplace.

Notice also that I haven't really shown any examples using exceptions, but trust
me when I say that things get even more complicated with those.

The points I am trying to make here are these:

1) There are many ways for control flow to leave a current context, either by
going down (calling a subroutine which never returns), by going up (through a
normal return) or by going out (an exception or a continuation). There are no
real mechanisms to guarantee an action is performed as soon as control flow
escapes from a context, because there isn't any real way to even detect that it
has escaped.

2) We can attach a cleanup handler to the context, and when the context is
collected we can force the resource to be collected too. Of course, unless
you've created and saved references to the resource, GC would have collected it
at the same time anyway, and then if you do have saved references somewhere
they are going to point to garbage data or worse. In the best case, things
happen exactly as they already happen. In the worst case, new segfaults.

We could expand the `using` block to look something like this:

    {
        (function(var bar) {
            var __exception = null;
            var __ctx = get_context();
            __ctx.dispose_on_escape_continuation(bar);
            try {
                do_something_with(bar);
            } catch(e) {
                __exception = e;
            }
            bar.dispose();
            bar = null;     // so lexicals can't find it
            if (__exception != null)
                rethrow(__exception);
        })(new Bar());
    }

That seems to cover most of the cases, if we can find a way to implement the
`__ctx.dispose_on_escape_continuation` method, which would be no trivial thing.
We would need to be able to tell that a continuation was invoked and that the
continuation was leading up the call chain above the `using` closure because
continuations happen all the time for things like return and the vast majority
of them are not control-flow-boggling escapes. If we can't implement such a
beast, maybe we say that the `using` structure just doesn't work well with
continuations and resumable exceptions and call it a day. Caveat Emptor.

We would love to be able to make this kind of thing work, and we would love to
have a feature like this. I can't really see a good way to do it, but I do smell
the beginnings of a research opportunity here, if there are any motivated
graduate students following along at home.
