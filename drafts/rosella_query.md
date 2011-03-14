A few days ago I introduced [Rosella][], a new library project I've been
working on for Parrot. Rosella is less a single solution or framework than
it is a collection of several smaller utility libraries. Each library helps
to enable either some best practices, provide a set of stepping-stone tools
to help improve developer productivity, and occasionally just provide some
cool stuff to make your code better.

recently I've added a new library to the Rosella umbrella, "Query".

The Rosella Query library has two main portions: The Query and Path classes.

## Rosella Query

The query library is, in part, my reimagining of the C# "System.Linq" library
for Parrot. Actually, it's a small reimagining of a very small portion of
Linq, but the inspiration is clear.

With `Rosella::Query` and related helper classes you have a few tools
available which make work with collections of things easier and more
familiar. This module provides three helper functions: map, filter, and
fold. These are analogs for the Linq functions `.Select()`, '.Where()`,
and `.Aggregate()`, respectively. The `Rosella::Query` module is basically
a facade over the underlying mechanism `Rosella::Query::Provider`.
`Provider` and its subclasses provide the actual implementations of these
search functions for different data types. We have `Provider::Array` and
`Provider::Hash` classes now, but other options could easily be added
later.

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

## Rosella Path

The Path library is similiar in the sense that you can request data from
an object. It uses xpath-like syntax for querying data from nested data
types. At the moment it is able to traverse hash keys and object attributes,
but other abilities are on the horizon. Here's a lengthy example of hash
key traversal:

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
put together such a large nested set of objects.

## Future

These few examples barely scratch the surface of what this new library
can do already, and I have plans for lots of awesome new features in the
future.
