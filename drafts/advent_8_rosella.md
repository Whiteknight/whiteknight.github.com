---
layout: post
categories: [Parrot, Advent2011]
title: Advent 8 - Rosella
---

I had already written a post about Rosella for my woefully inadequate advent
calendar, but it got lost to the sands of time (I think it's on my other
computer, committed but not pushed). Considering that I should have been on
schedule to be on post #21 or #22 by now but am instead only on #8, I can't
really afford to not write a post when I have the ideas inside me. This post is
also going to be about Rosella, and if I ever find the first one I may post that
as well. I clearly haven't had enough spare writing time this month to be even
remotely picky about what I post or when.

I'm thinking I might like to try this little experiment again later in the year,
when I've had more time to prepare and have fewer other things in real life
demanding my unidivided attention. Maybe we'll shoot for some kind of
christmas-in-July thing. Until then, I'll happily hand the "best Advent
Calendar" crown back to moritz and the Perl peoples.

Rosella is a library project I started as a way to let me work on lots of
different project ideas I had without having to duplicate build and test
infrastructure. The goals from the project were solidified pretty early in the
project lifecycle and have remained unchanged ever since: To provide solutions
to common developer problems, in pure Parrot, in ways that are portable,
reusable, and configurable. I've already talked about three of the oldest and
most important libraries in the set: Test, Harness and MockObject on Day 5 of
this Advent calendar. My other written-but-not-published post talked about three
additional libraries as well: Query, FileSystem and Proxy. To avoid much
duplicate effort in case I do ever find that post again, I'll write about a few
different libraries: Container, Random and Template.

The Container library is one of the oldest libraries in Rosella. It is an
implementation of a dependency injection container for Parrot, and originally
lived in it's own Parrot-Container repository on github. My other library, for
unit testing was also a separate repo. One day I was thinking about how I might
like to use the Parrot-Container project to manage dependency injection in the
Harness library, and trying to figure out how I wanted to manage the software
dependencies. I wanted to use the Container library in the implementation of the
unit test Harness, but I wanted to use the Harness and Test libraries to
implement the test suite for the Container library repository. Combine that with
a bunch of other ideas for new libraries I had brewing in the back of my mind
and Rosella was quickly born.

The Container library is not quite as easy to use as a counter-part in the .NET
or JVM environments because things are not always strongly typed, especially not
at compile time or early in runtime when the container is initializing but the
various bits of initialization logic in the program have not yet run. The Rakudo
folks with their fancy object system and notion of gradual typing might have a
different idea, but in terms of pure-parrot code, as Parrot exists right now,
we can't query a Sub PMC and ask it what types of parameters it expects to
receive. For people who might be familiar with other dependency injection (or
"inversion of control") systems like Unity or Ninject, the syntax for setting up
Rosella's container type might seem a little verbose. It's plenty powerful (and
I have plenty of plans to upgrade things as Parrot's threading support and
object system improve in 2012 and beyond), but it is verbose:

    var container = new Rosella.Container();
    container
        .register(class Foo)
        .register(class Bar,
            new Rosella.Container.Resolver.TypeConstructor(class Bar, "Bar", 1, 2, 3)
            new Rosella.Container.LifetimeManager.Permanent()
        )
        .register(class Baz,
            new Rosella.Container.Resolver.TypeConstructor(class Baz, "BUILD"
                new Rosella.Container.Argument.Resolve(class Foo),
                new Rosella.Container.Argument.Resolve(class Bar)
            )
            new Rosella.Container.Option.Attribute("my_attr", "value"),
            new Rosella.Container.LifetimeManager.Thread()
        )
        .alias(class Baz, "Baz");

The part that is significantly more verbose here than you would expect in Unity
for example is the part where I have to specify the types of arguments to pass
to the constructor. In a platform like .NET, the container can read the type
metadata from the constructor object itself and make that decision. In Parrot,
since we currently don't make that kind of information available, you must
specify. In Rosella's offering you do have many of the features that you would
expect from one of the more well-known containers: the ability to specify
registration lifetimes, the ability to initialize objects by making method calls
and setting attribute values after construction, the ability to specify global
singleton instances, etc. It is a pretty cool tool, but I haven't yet made as
much use of it as I would have liked. In 2012 I expect to start integrating the
container library into some of the other libraries, such as Harness, CommandLine
and Template, to make user configuration easier and more straight-forward.

The Random library is a relative newcomer to Rosella, but is already
demonstrating its usefulness in a variety of ways. The Random library was born
from a few ideas I had turned into GCI tasks. An intrepid young student wrote
an implementation of the Mersenne Twister algorithm for me in Winxed, and I set
about writing up several other components such as a Box-Muller transform, a UUID
generator, a Fisher-Yates array shuffler and a few other things. Now, you can do
cool things with random numbers:

    var prng = Rosella.Random.default_uniform_random()
    int r = prng.get_int();

    var uuid = Rosella.Random.UUID.new_uuid();
    string id = string(uuid);

    var a = [1, 2, 3, 4, 5, 6, 7, 8];
    Rosella.Random.shuffle_array(a);

Implementations of things aren't all perfect and there are a handful of bugs to
be worked out (especially bugs resulting from arithmetic differences between
32-bit and 64-bit machines) but it is already very usable and reliable for the
most part. If you need a random number generator, or a UUID generator, or other
random-related things, this is a very nice tool to have available.

The Template library is something I wanted to write for a long time, but had to
wait until all the prerequisites were in place first. Template is a
text-templating engine library. You create an Engine object and feed it two
basic pieces of information: The string template to use and the data context
object to fill in the blanks. Presto-chango, out comes the complete text.

The Template engine can execute basic logical operations depending on the values
in the data context, it can compile and execute inline snippets of code, it can
load and assemble pieces of template from separate files, and do several other
things that you would expect a templating library to do. I could write many
examples of templates and their use, but I'll stick with only one small example
here for brevity:

Template:

    Let's learn about <%
        var bar = context["animals"];
        return elements(bar);
    %> new animals!
    <$ for sound in animals $>
        The <# __KEY__ #> says <# sound #>
    <$ endfor $>

Code:

    string template = ...;
    var context = {
        "animals" : {
            "Cow" : "Moo",
            "Bear" : "Growl!",
            "Cat" : "I CAN HAZ CHEESEBURGER"
        }
    };
    var engine = new Rosella.Template.Engine();
    string output = engine.generate(template, context);

And the end result would be something like this (minus hash ordering concerns):

    Let's learn about 3 new animals!
    The Cow says Moo
    The Bear says Growl!
    The Cat says I CAN HAZ CHEESEBURGER

Rosella eats plenty of it's own dogfood here. Several test files in the Rosella
test suite and a few boiler-plate source code files are generated using the
Rosella Template library. I'm planning to start using it to generate skeleton
files for Rosella documentation as well. In the future, I may also use it to
help with generating content for this very blog!

In 2012 Rosella is going to be adding a bunch of cool new stuff. Considering how
far Rosella has come by now and the fact that it's less than a year old, it's
kind of hard to speculate where we will be next year around this time. I'm
planning to add a new Date/Time library within the next few weeks. I'm also
planning a new reflection/packfile library, a benchmarking library, a code
assertions library, and rewrites to several of the existing libraries to add
new functionality and optimize performance in some key ways. Those are only my
plans for the first two or three months of 2012!




