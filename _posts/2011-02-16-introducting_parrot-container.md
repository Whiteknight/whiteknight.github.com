---
layout: post
categories: [Parrot, ParrotContainer]
title: Introducing Parrot-Container
---

At my day job I write web-related software in C#. [I like C#, despite some of
its problems][me_on_csharp]. In fact, it is my favorite language for general
development on Windows machines.

[me_on_csharp]: /2010/04/06/problems_with_c.html

One thing that I particularly like about my C# coding projects is [Unity][].
Unity is a dependency injection framework which can provide your software a
number of benefits: improved decoupling, dependency injection, dependency
identification, ready-made factory and singleton patterning, recursive object
resolution, avoidance of global variables, easy system configuration, etc. It
is a really great tool and where I've used Unity I've found the quality of the
resulting code to be much higher than without it.

[Unity]: http://unity.codeplex.com/

For some time, I've wished we had something similar for Parrot, but it wasn't
until yesterday evening when I started putting down some exploratory code to
see if it would even be possible to do. It turns out that it does make sense,
if we change a few ideas around. This morning, I pushed an iniatial set of
classes to github for the new [Parrot-Container][parrot_container] project.

[parrot_container]: https://github.com/Whiteknight/parrot-container

First, let me talk about what Unity does and what dependency injection is.
Then I'll show some example code from Parrot-Container and talk about how it
can be used and what my development plans are for it.

What we can do with a Unity Container object is register types and objects
under a certain type key. Later when I ask Unity for an object of a specific
type, it can provide it to me from the catalog of registrations. If I register
a type, it will create me a fresh new object of that type. If I register an
instance, it will return me the pre-existing instance. What's great is that I
can register a concrete type with a key that is an interface or an abstract
type, and rely on unity to deliver me an object of the configured subclass.
Here's some example code:

{% highlight csharp %}

UnityContainer SetupContainer() {
    UnityContainer container = new UnityContainer();
    container.RegisterType(typeof(IFoo), typeof(Foo));
    IFoo foo = container.Resolve(typeof(IFoo));
    container.RegisterInstance(typeof(IBar), new Bar("Whatever!"));
    IBar bar = container.Resolve(typeof(IBar));
}

{% endhighlight %}

Actually, through the magic of type generics in C#, we can write the code much
more cleanly:

{% highlight csharp %}

container.RegisterType<IFoo, Foo>();
IFoo foo = container.Resolve<IFoo>();

{% endhighlight %}

This is a nice feature, but not entirely powerful. Where Unity really starts
to become valuable is in handling constructor parameters.

{% highlight csharp %}

class Program {
    public static void Main(string[] args) {
        UnityContainer container = new UnityContainer();
        container.RegisterInstance(typeof(IFoo), new Foo("Whatever!"));
        container.RegisterType(typeof(IBar), typeof(Bar));
        IBar bar = container.Resolve(typeof(IBar));
    }
}

class Bar {
    public Bar(IFoo foo) { ... }
}

{% endhighlight %}

Notice how the class `Bar` requires an `IFoo` in its constructor? Notice how
we didn't specify one to pass when resolving the new `bar` variable? Unity
reads the constructor signature and *automatically resolves instances of all
variable types asked for*. Anywhere in my program, I can ask for a fresh new
`IBar` object without having to know anything about what the dependencies are
of the particular concrete class which is being constructed. I don't need to
know what the dependencies of `Bar` are when I just want any old `IBar`
object. I ask for an `IBar`, and Unity automatically injects the necessary
dependecies for me.

In Parrot, I would really like to be able to do something similar. I would
like to be able to ask for objects and be able to ignore, in most cases, the
messy details about how to instantiate and initialize them. This is especially
important since there really is no standard way of initializing a new Parrot
object with invariant data. If I have a regular Parrot type, like a built-in
PMC type or a user-defined PIR Class PMC, I can pass in a single PMC
initializer when I create it. For built-in types this logic is handled by
`VTABLE_init_pmc`. For user-defined classes it would happen through
`VTABLE_instantiate`. If we assume for the sake of argument that most
initializer PMCs like this would be arrays or hashes of named quantities, we
do have a relatively flexible way of creating new objects.

However, if I'm using a P6protoobject things are a little bit messier. In
the P6protoobject implementation in the Parrot repo, there is a `.new()`
method which can be used to create a new object of the specified type, but
that `.new()` method doesn't take any parameters, and doesn't call either of
the constructor-ish methods described in the Perl6 spec: CREATE and BUILD.

Even if we override the `instantiate` vtable in our custom class, there's no
way through the normal P6protoobject interface to pass an initializer value to
it. There are ways around this, of course. We could update the `.new()` method
in P6protoobject to allow initializers, or to call a BUILD method if it
exists. We could override that method with custom behavior in each of our own
projects, although that's a big messy pain in the ass. We could simply
instantiate every NQP object using a two-stage system:

    my $foo := My::Foo.new();
    $foo.initialize(args);

Kakapo got around this issue by overriding `P6protoobject.new()` with a
custom implementation which called a custom `_init_obj()` method. That's not
in the Perl6 spec, although it would have been trivial to rename the method
which was called to "BUILD" for compliance. But even with that overridden
constructor logic, it would only work on P6protoobjects, not on ordinary
Parrot Class PMCs, PMCProxy PMCs, or even some other kind of exotic metaobject
that the user was using. Either we would need to take substantial effort to
wrap all the non-P6ish types with P6metaclass wrappers, or we would need to
differentiate between types in the application code. A large part of Kakapo
involved wrapping built-in types and adding all sorts of add-on logic,
improved constructor logic, extension methods, etc. An equally large part of
Kakapo also involved handling dependencies between classes, so that we were
setting up all this scaffolding in specific orders so we didn't run into
conflicts.

Parrot-Container aims to change all of that. We register objects by "type",
and can resolve objects using a specified sequence of initialization steps.
This way the user of an object doesn't need to be aware of the initialization
steps and dependencies of the objects it creates. Parrot-Container's Container
class does all that for you. I put the word "type" in quotes above because I
really don't key on types, but instead key on stringified slugs. The key can
be a P6protoobject, a P6metaclass, a Parrot Class PMC, a String, a string
array PMC, a NameSpace PMC, a Key PMC, or anything else that can be
stringified. Because we are stringifying, it basically is a slug-based lookup
system, and we would be able to register types and instances by name instead
of by type.

When I register a type, I can register it with a list of initialization
actions. Those actions can involve passing in an initialization PMC to
`VTABLE_init_pmc` or `VTABLE_instantiate`, or calling a method with a
specified list of parameters, or calling a sub which takes the new object and
a list of parameters, or calling a completely custom factory method to
generate the new object. Without further adieu, here is an example of code
that *works right now* in Parrot-Container:

    INIT { pir::load_bytecode("parrot_container.pbc"); }
    my $c := ParrotContainer::build(ParrotContainer::Container)

    $c.register_type("String", :meth_inits([
        ParrotContainer::build(ParrotContainer::Initializer::Sub,
            sub ($obj) {
                pir::set__vPS($obj, "FooBarBaz");
            }, []
        ),
        ParrotContainer::build(ParrotContainer::Initializer::Method,
            "replace", [
                ParrotContainer::build(ParrotContainer::InitializerArg::Instance, "B", :position(0)),
                ParrotContainer::build(ParrotContainer::InitializerArg::Instance, "C", :position(1))
            ]
        )
    ]));
    my $bar := $c.resolve("String");
    pir::say($bar); # "FooCarCaz"

The `ParrotContainer::build()` function is a convenience wrapper that
instantiates a new object from the type (P6protoobject or anything else that
represents a type) and automatically invokes a BUILD method with the provided
arguments, if any BUILD method exists in that class. By using
`ParrotContainer::build()` faithfully, we can use "BUILD" constructors with
any type we create, whether it be P6object-based or not.

Since we're using dynamically-typed languages on Parrot, we can't really read
constructor parameter lists and extract a list of required types from that.
For this reason we need to do a little bit more work up-front when we register
types with the container to specify injection parameters. If we want the
Container to automatically resolve parameters recursively, we can use a
`ParrotContainer::InitializerArg::Resolve` type instead. This will ask the
container to resolve the parameter by tag name dynamically.

In the sequence above, we register the lowly String type with two
initialization routines. The first is a nested sub which sets the value to
`"FooBarBaz"`. The second routine calls the `.replace()` method with two
positional arguments to replace all instances of "B" with "C". When we resolve
the "String" type, we get a fresh String PMC with the value "FooCarCaz", which
we expect. Here's another example:

    $c.register_type("Complex", :init_pmc("1+2i"));
    my $complex := $c.resolve("Complex");
    pir::say($complex);    # "1+2i"

We can manually specify an initialization PMC to use, which is passed to the
`VTABLE_init_pmc`. In the case of Complex, it takes a String argument and
parses out a complex value from it.

There is one other feature that I've built into ParrotContainer, and which
I've also been putting together in a separate lighter-weight class as well:
Prototype-based object building. In a prototype system, we register a
prototype object with the container, and when we resolve that prototype's
type, we get a clone of it. This is a necessary feature for JavaScript, and
Parrot-Container provides it out of the box:

    my $proto := new__PPP("Complex", "1+2i");
    $c.register_prototype("Complex", $proto);
    my $complex := $c.resolve("Complex");
    pir::say($complex);     # "1+2i"
    $complex := "4+5i";
    pir::say($complex);     # "4+5i"
    pir::say($proto);       # "1+2i"

With prototype we can also specify a number of initializer actions, which
could be used to resolve child properties if deep cloning or other custom
per-instance modification is required. Notice that, like in JavaScript, if we
change the prototype after it's been registered, future clones of it will
have the new properties (although existing clones will not). I'm also writing
a much smaller `PrototypeManager` class which will handle prototype-based
activities like this without all the features and overhead of the Container
class.

That's a basic overview of the new Parrot-Container project and what it is
currently capable of. I have a lot of development I want to be doing on it
in the future to add new features and abilities. I already have a number of
planned uses for it, and I would be interested to hear what other people think
about it as well.


