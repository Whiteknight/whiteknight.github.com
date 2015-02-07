---
layout: post
categories: [Parrot, Rosella, Reflect]
title: Rosella Reflect
---

Earlier this month I released the new Reflect library in Rosella. I hadn't
mentioned it before, but the library is sufficiently interesting that I want
to talk about it at least a little bit. The Reflect library adds in tools for
reflection. Somewhere, an etymologist weeps a tear of joy for the creative
naming, I'm sure.

The Reflect library adds in wrappers for classes and packfiles that makes them
easier to work with for many operations. First, I'd like to use a couple code
examples to show the most basic API:

    // Get the Sub PMC that we're currently executing
    var s = Rosella.Reflect.get_current_sub();

    // Get the current context
    var c = Rosella.Reflect.get_current_context();

    // Get the current object, if the current Sub is a method call
    var obj = Rosella.Reflect.get_current_object();

    // Get the class of the current object, if the current Sub is a method call
    var c = Rosella.Reflect.get_current_class();

    // Get a Module object for the packfile where the current Sub is defined
    var m = Rosella.Reflect.get_current_module();

    // Get a reflection wrapper object for the given Parrot Class PMC
    var r = Rosella.Reflect.get_class_reflector(myClass);

    // Get a Module object for the packfile in "foo/bar.pbc", loading it as
    // necessary
    var m = Rosella.Reflect.Module.load("foo/bar.pbc");

That's the basic API that the library provides to get basic information about
where execution is happening at the moment when the call is made. Once you
have a Module object or a Class reflector object, you can do all sorts of cool
things that used to be a pain in the butt to do manually:

    var m = Rosella.Reflect.get_current_module();
    say(m);          // Stringified, produces the name and version of the packfile
    m.load();        // Execute all :tag("load") and :load functions
    n.init();        // Execute all :tag("init") and :init functions
    say(m.version(); // Get the version string of the packfile "X.Y.Z"
    say(m.path());   // The on-disk path to the current packfile

    // Get a hash of all Class PMCs defined at compile-time (using the :method
    // flag on Subs) defined in the packfile, keyed by name
    var c = m.classes();

    // Get a list of all non-:anon functions defined in the packfile
    var f = m.functions();

    // Get a hash of all non-:anon functions in the packfile, organized into
    // a hash keyed by namespace
    var f = m.functions_by_ns();

    // Get a hash of all NameSpace PMCs defined at compile-time
    var ns = m.namespaces();

Once you have Class and NameSpace PMCs from the packfile, you can start to
do all sorts of cool operations and analyses on them. Once you have a Class
reflector object, you can do even more stuff with that:

    var c = Rosella.Reflect.get_current_class();

    // Create a new object of the current type
    var o = c.new();

    // Say the name of the class
    say(c.name());

    // Attributes are encapsulated as objects. You can get an Attribute
    // reflector and use it later to get and set values on objects of this
    // type or subclasses
    var attr = c.get_attr("foo");
    var value = attr.get_value(o);
    attr.set_value(o, "whatever");

    // Methods are also encapsulated. You can get a method reflector now and
    // invoke it on objects later (including objects of different types)
    var method = c.get_method("bar");
    var result = method.invoke(o);
    var meths = c.get_all_methods();

    // Basic capability detection. Determine if objects are members of the
    // class or their subsets, and determine if the class can perform certain
    // methods
    if (c.isa(o)) { ... }
    if (c.can("bar")) { ... }

I hope the code examples make up for the terse explanations.

The Reflect library is currently focused on reading data from things like
Classes and Packfiles, not on creating these things like the new PACT project
is supposed to do. I want to extend this library even further with abilities
to further introspect functions down to the opcode level and then...Well, when
we have a stream of opcodes to analyze the possibilities are endless. I'd also
like the ability to get better introspection of the interpreter and global
state, though a cleaner interface than the hodge-podge of `interpinfo` opcodes
and ParrotInterpreter PMC methods and whatever else we currently use.

As always, using the interface Rosella provides will help to insulate you from
changes to the various underlying mechanisms when we finally get around to
cleaning them up and making them sane. There isn't a huge push to make such
cleanups on a large scale yet, but I wouldn't be surprised if a few things
started getting prettified in the coming months at a slow pace.

I've already started using the new library in several of the Rosella utility
programs such as those that create a winxed header file or a test suite from
an existing packfile. In all cases the updated programs are both cleaner and
have more functionality than the previous incarnations. Expect to see this
library improve and grow in 2012 and beyond, and expect to see it work closely
with PACT, once that project gets moving forward.


