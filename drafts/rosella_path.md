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
