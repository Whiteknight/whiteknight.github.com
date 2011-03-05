---
layout: post
categories: [Parrot, Rosella]
title: Proxies and Mocks with Rosella
---

Late last week I finaly got to work on two projects I had wanted to start for
a while: a proxy library and a mock object library. Both of these are parts of
[Rosella][]. Today I'm going to talk a little bit about what each of these
are, how they are used, and why it's so cool to have them available.

[Rosella]: http://github.com/Whiteknight/Rosella

## Proxies

[Proxies][proxy_pattern] are objects that you can use in place of another
object. You can replace the real "target" object with the "proxy" object, and
use the proxy in places where you were previously using the target. There are
several different reasons why you might want to use a stand-in object in place
of the real thing. Here are just a small number of uses. Each of these uses
the same general idea of proxying, but may have a slightly different
implementations of it:

[proxy_pattern]: http://en.wikipedia.org/wiki/Proxy_pattern

1. The target object is extremely expensive to create and we want to only
   create it lazily, on demand. In these cases we create a cheap proxy, pass
   that around until the target is needed, then use the proxy to lazily create
   the target and redirect all property and method accesses to the target.
2. The target object is located remotely, such as in a different process or
   on a completely different computer. In the client-server case, the client
   would have a proxy object for a target object on the server. Accesses on
   the proxy object are turned into remote calls over the network, which
   eventually reach the target object.
3. The target object is expensive to use, but its generated results are
   extremely reproducible. The proxy object can be used in place of the target
   object and can provide result caching. In some circles this would be known
   as "memoization".
4. The target object does not exist at all, but can be simulated in short
   sequences by a proxy.

There are more uses, but these are some pretty big and important ones.

Rosella's Proxy library uses a modular architecture to produce proxy objects
for a variety of uses. Every proxy object has an associated controller object.
Depending on how the proxy is setup, various method and vtable calls can be
transparently redirected to the controller. In this way the proxy object can
have a clean interface and can transparently appear to be of the target type,
while the controller can implement all the control and manipulation logic.

Here's a short example snippet to show how proxies can be used in Rosella:

    class FooController is Rosella::Proxy::Controller {
        method find_method($proxy, $name) {
            return method($arg) { return "fake $arg"; }
        }
    }

    class My::Foo {
        method bar($arg) { return "real $arg"; }
    }

    my $f := My::Foo.new();
    my $result := $f.bar("test 1");
    pir::say($result);          # "real test 1"

    my $factory := Rosella::build(Rosella::Proxy::Factory, My::Foo, [
        Rosella::build(Rosella::Proxy::Builder::MethodIntercept)
    ]);

    my $p := $factory.get_proxy(FooController.new());
    $result := $p.bar("test 2");
    pir::say($result);          # "fake test 2"

The `Rosella::Proxy::Factory` class creates proxy objects. By default, the
proxies created are pretty dull. The don't handle most vtable or method
accesses. This is pretty boring. However, we can use a series of builder
objects to add additional functionality to our proxies to make them more
interesting as needed. Currently I have three types of builders:
`Rosella::Proxy::Builder::MethodIntercept` which allows us to intercept
method calls and pass them to the controller,
`Rosella::Proxy::Builder::AttributeIntercept` allows us to intercept get and
set attribute requests, and `Rosella::Proxy::Builder::Imitate` which
interceps calls to VTABLE_does (and eventually will intercept "isa", and
"can", when and if Parrot allows those things to be overridden).

Builders tell the proxy to intercept certain requests and redirect them to the
controller. By creating your own builders and controllers, you can implement
all sorts of interesting behavior for proxies. For example, here is a short
example I've thrown together just now to implement memoization on proxied
subs:

    INIT { pir::load_bytecode("rosella/mockobject.pbc"); }

    class MemoizerController is Rosella::Proxy::Controller {
        has %!cache;
        has &!block;
        method BUILD(&block) { %!cache := {}; &!block := &block; }

        method invoke($proxy, @pos, %named)
        {
            my $key := pir::sprintf__SSP('%d,%d,%d', @pos);
            my $result := %!cache{$key};
            if pir::defined($result) {
                pir::say("Found cached result!");
                return $result;
            }
            $result := &!block(|@pos, |%named);
            %!cache{$key} := $result;
            return $result;
        }
    }

    my &block := sub($a, $b, $c) { return $a + $b * $c; }
    my $factory := Rosella::build(Rosella::Proxy::Factory, pir::typeof__PP(&block), [
        Rosella::build(Rosella::Proxy::Builder::InvokeIntercept)
    ]);
    my $p := $factory.get_proxy(Rosella::build(MemoizerController, &block));
    $p(1, 2, 3);
    $p(1, 2, 3);
    $p(1, 2, 3);
    $p(1, 2, 3);

This is just a short example, but if you run it with current Rosella you will
see that indeed it does perform transparent memoization as expected. I haven't
done any benchmarking so I can't say how expensive this mechanism is and
therefore how expensive a sub must be before such a memoization tool would
lead to usable savings. I've thought about turning this into a separate
memoization library, but I suspect I would need to add a lot of code to make
this mechanism general enough for common use. That extra machinery would just
make this system even slower and therefore less useful. Regardless, this
little example does show the power of this proxy library.

One other particularly interesting use of proxies is, to me, the creation of
mock objects for testing.

## Mock Objects

The theory of [mock objects][] is that you create a stand-in (proxy) object
and use that to test the behaviors of separate system. In a mock-object test,
you set up the mock object (or, simply "mock") with a number of expectations.
On each request, such as a method or attribute access, you look up to see if
there is a matching expectation. If there isn't, it's a failure. If there is,
it's a match. At the end of the test we should have no unexpected accesses,
and all of our expectations should have been matched.

[mock_objects]: http://en.wikipedia.org/wiki/Mock_Object

What mock objects allow us to do is test not only the external interfaces for
an object, but also the internal interfaces as well. I can test my calls
*into* the object, and I can all test its calls back out. This allows us to
do extremely robust black-box and end-to-end testing of a single system.

Rosella's mock object system is inspired, at least in part, by the NMock
library for .NET.

Without further adieu, let's take a look at an example test using Rosella's
test libary and mock object library.

    INIT {
        pir::load_bytecode("rosella/test.pbc");
        pir::load_bytecode("rosella/mockobject.pbc");
    }

    class MyClass { }   # Class to mock. For now it can be empty

    Rosella::Test::test(MyTest);
    class MyTest is Rosella::Test::Testcase {
        method test_one_method_call() {
            my $f := Rosella::build(Rosella::MockObject::Factory, MyClass);
            my $c := $f.get_mock_controller();
            $c.expect().at_least(2).method("test").with_args(1, 2, 3);
            my $m := $c.mock();
            $m.test(1, 2, 3); # Passes, we expect it
            $m.test(2, 3, 4); # Fails, wrong args
            $m.test(1, 2, 3); # Passes again
            $c.verify();      # Check that we have no unmet expectations
        }
    }

Rosella's mocks are already pretty versatile, and I have new features that I
want to add in the future. Here are some things you can do with Rosella's
MockObject libary *right now*:

    # Expect exactly one call to "test", any args. The mock will return
    # the value "ok" to the caller.
    $c.expect().once().method("test").with_any_args().will_return("ok");

    # Expect that we never read the value of attribute "foo"
    $c.expect().none().get_attribute("foo");

    # Expect that we read the attribute "length" exactly once. It will return
    # the value 7.
    $c.expect().once().get_attribute("length").will_return(7);

    # Expect that we set the value of attribute "foo" to the value "baz"
    # at most 3 times. This could be 0, 1, 2, or 3 times.
    $c.expect().at_most(3).set_attribute("bar").with_args("baz");

    # Expect that we call the method "next" exactly once. The mock will then
    # throw an exception. We can use .will_throw() to test error-handling
    # capabilities of our code.
    $c.expect().once().method("next").will_throw("whoops!");

    # Expect that this proxy is invoked like a Sub, with the arguments
    # 1, 2, 3, and will return the value 4. (requires a flag to be set in the
    # proxy factory for this to work)
    $c.expect().once().invoke().with_args(1, 2, 3).will_return(4);

If this were a mock object library for a different VM, I could probably
declare what I have now to be feature completeness. However, since this is
Parrot and it has some cool functionality not found too often in other VMs,
I have more features to add. Specifically I need to add the ability to
intercept various other vtables, so that a mock can convincingly look like
primitive types (Integer, String), and aggregates (ResizablePMCArray, Hash).

Mocks start to become extremely powerful when you combine them with a
dependency injection container. If the code you are testing uses the container
to resolve objects, you can silently add mocks to the container for testing.
When you run your code for your test, it automatically receives mocks from
the container instead of the "normal" objects, and we can start testing the
internal algorithms and pathways of our system directly. We do not need to
duplicate our algorithms and sequences in our tests, and risk the tests
getting out of date with the code. Instead, we can test the production code
in-place, exactly as it is used in our program. Lucky for everybody involved,
Rosella provides just such a [Container][] type.

[Container]: /2011/02/16/introducting_parrot-container.html

## Rosella Release

I'm getting to be pretty happy with the list of features provided by Rosella,
and I'm starting to think about preparing a stable release of the software.
I suspect I will try to target a release against Parrot 3.2, after that comes
out.
