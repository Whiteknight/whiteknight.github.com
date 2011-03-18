A few days ago I introduced [Rosella][], a new library project I've been
working on for Parrot. Rosella is less a single solution or framework than
it is a collection of several smaller utility libraries. Each library helps
to enable either some best practices, provide a set of stepping-stone tools
to help improve developer productivity, and occasionally just provide some
cool stuff to make your code better.

[Rosella]: https://github.com/Whiteknight/Rosella

Recently I've added a new library to the Rosella umbrella, "Query". Query,
as it's currently conceived, is a library for working with aggregate objects.
it's still pretty experimental and is in need of a little bit more design
work and refactoring, but it's already become pretty usable and I have high
hopes for it in the future.

The Rosella Query library currently has two main portions: The Query and Path
classes. These may be broken out into their own libraries eventually, they are
only loosely related to each other through the mantra of "working with
aggregate objects".

## Rosella Query

The Query class is, in part, my reimagining of the C# "System.Linq" library
for Parrot. Actually, it's a small reimagining of a very small portion of
Linq, but the inspiration is clear.

With `Rosella::Query` and related helper classes you have a few tools
available which make work with collections of things easier and more
familiar. This module provides some "higher-order" helper functions that
people may be familiar with: map, filter, and fold for example. These are
analogs for the Linq functions `.Select()`, '.Where()`, and `.Aggregate()`,
respectively. The `Rosella::Query` module is basically a
[facade][facade_pattern] over the underlying mechanism
`Rosella::Query::Provider`.

[facade_pattern]: http://en.wikipedia.org/wiki/Facade_pattern

`Provider` and its subclasses provide the actual implementations of these
search functions for different data types. We have `Provider::Array` and
`Provider::Hash` classes now, but other options could easily be added
later. Some additional providers that I can see a clear need for are things
like `Provider::XML` for searching through XML streams, or `Provider::PAST`
for searching around through an HLL compilers PAST tree.

So what can you do with Query? Right now the syntax is a little clunky, but
I intend to provide a mechanism to inject the methods into foreign namespaces
so you can use these tools a little bit more gracefully. At the moment, here
are some examples:

    my @data := <foo bar baz fie>;
    my @new_data := Rosella::Query::map(@data,
        sub($item) { return "($item)"; }
    );
    for @new_data { pir::say($_); }

That quick example outputs a new list where all the contents of the input
list are stringified and surrounded by parenthesis. Trivial, but it gets
the point across.

    my %data := {};
    %data{"foo"} := 1;
    %data{"bar"} := 2;
    %data{"baz"} := 3;
    %data{"fie"} := 4;
    my %new_data := Rosella::Query::filter(%data,
        sub($item) {
            return $item % 2 == 0;
        }
    );

Here's an example using filter on a hash. We filter out all values that
aren't even numbers and return a new hash with only the remaining items.
As expected, the new hash has only two elements "bar" and "fie".

    my @data := <1 2 3 4>;
    my $new_data := Rosella::Query::fold(@data,
        sub ($sum, $item) { return $sum + $item; },
        :seed(20)
    );

In this case, we do a running sum of all elements in the array, starting
with the seed value of 20. As anybody would expect, 20+1+2+3+4 = 30.

The Query class portions of the library need to grow in two ways: First
adding more standard interface methods for the providers to implement, and
second adding more providers to satisfy the interface for new types of
aggregate data sets.

## Rosella Path

The Path class is similiar in the sense that you can request data from
an aggregate object. It uses xpath-like syntax for querying data from nested
aggregates, where the Query class instead focuses on common iteration patterns
for flat aggregates. At the moment it is able to traverse hash keys and object
attributes, but other abilities are on the horizon. Here's a lengthy example
of hash key traversal:

    my sub new_hash(*%h) { %h; }

    my $q := Rosella::build(Rosella::Query::Path);
    my %a := new_hash(
        :d(new_hash(
            :e(new_hash(
                :h("i")
            ))
        ))
    );
    %a{"d.e"} := new_hash(:f("g"));

    pir::say($q.get(%a, 'd.e.f'));
    pir::say($q.get(%a, 'd.e.h'));

The majority of the code in this example is used to construct a nested hash
`%a`. The really interesting bit is at the very bottom with the `.get()`
method. This method takes a path string and searches through the aggregate
to satisfy it. In the case of hash keys the search is longest-key first.
With the given hash, the following two lines are equivalent accesses:

    my $result := $q.get(%a, 'd.e.f');
    my $result := %a{"d.e"}{"f"};

And these two:

    my $result := $q.get(%a, 'd.e.h');
    my $result := %a{"d"}{"e"}{"h"};

You're not necessarily saving yourself any keystrokes with this library
(at least not in these two toy examples), but you are abstracting away
the storage structure and allowing a simple search path to return the
correct value. Finally, consider this to really understand the power of
the new library:

    my $result := $q.get($obj, 'foo.$!payload.bar.%!props.baz');

That example works *right now* in the library, if you take the time to
put together such a large nested set of objects. Keep in mind that the items
separated by periods can transparently be hash keys or attribute names. The
library starts searching from longest identifiers first, then slowly starts
whittling down until it finds a match. You could, for instance, have a hash
object with a single long key:

    my %data := {};
    %data{'foo.$!payload.bar.%!props.baz'} := "hello!";

...or you could have any other combination nested however you want it. The
biggest benefit, as I mentioned earlier, is that we gain the ability to
separate the data being consumed from the actual structure of the model which
provides that data.

To get an idea of where I am envisioning this functionality to go, consider
the idea of a text templating engine for Parrot similar to [Liquid][]. Given a
Template object which combines both a raw text string and a data context
object, the parser would read through the text string until it found
templating instructions, and could then pass off data requests to the Path
object for resolution.

[Liquid]: http://www.liquidmarkup.org/

## Future

These few examples barely scratch the surface of what this new library
and its two classes can do already. I have plans for lots of awesome new
features in the future, and would love to get feedback from potential users
about what to add and what should be changed.
