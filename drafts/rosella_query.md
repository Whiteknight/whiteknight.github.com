I spent a few days without internet access, and so I took all the energy I
normally spent surfing the internet and instead channeled it into Rosella.
Because of the effort, I've been able to refactor an older library that I've
been playing with and bring it up to proper, stable status. This new library,
"Query" is a library for higher-order functions over aggregate objects. It's
been heavily influenced by the System.Linq library from the world of .NET.
People who have .NET experience should be familar with all the concepts and
techniques in the Query library.

To start working with the Query libray, we need to take an object we want to
play with and wrap it up in a `Queryable`. For .NET fans out there, Queryable
is like a combination of `IEnumerable` with the various Linq extension methods
on it already.

    my @data := [1, 2, 3, 4];
    my $q := Rosella::Query::as_queryable(@data);

Once we have the Queryable, we can start running methods on it. Most methods
on Queryable return a new Queryable, so we can chain result sets. Here's a
quick example taken out of the test suite:

    my @data := [1, 2, 3, 4, 5, 6, 7, 8, 9];
    my $sum := Rosella::Query::as_queryable(@data)
        .map(sub($i) { return $i * $i; })
        .filter(sub($j) { return $j % 2; })
        .fold(fsub($s, $i) { return $s + $i; })
        .data();
    pir::say($sum); # 165

One departure I have made from the Linq library is to use the more "classic"
names for the various operations. Methods `.Select()`, '.Where()`, and
`.Aggregate()` are called "map", "filter", and "fold" respectively. In this
example above we start with an array of integers from 1 to 9. We square each
element with `map`, use `filter` to remove the even elements, use `fold` to
sum them all together, and finally we use the helper method `.data()` to
get access to the internal result data of the Queryable. The final result is
`1 + 9 + 25 + 49 + 81 = 165`, as we expect.

The `Rosella::Query::Queryable` class is basically a [facade][facade_pattern]
over the underlying mechanism `Rosella::Query::Provider` and subclasses. The
Provider types provide the Query behaviors for different data types. Right now
I have one for Array, Hash, and Scalar. Others can be used instead, if you
want to provide a custom subclass. The necessary interface is a little bit
larger than I would like right now, but it's not impossible to provide your
own custom type if you want to.

[facade_pattern]: http://en.wikipedia.org/wiki/Facade_pattern

Other features that I've added are the abilities to convert data types to
different output formats easily. Here's an example where we convert an array
to a hash:

    my @data := [1, 2, 3, 4];
    my %hash := Rosella::Query::as_queryable(@data)
        .to_hash(sub($i) { return "Square $i"; })
        .map(sub($i) { return $i * $i; })
        .data;

The function reference to `.to_hash()` takes each element from the array and
is expected to return a string key for inserting it into the hash. We start
out with an array `[1, 2, 3, 4]`, and end up with a hash:

    %hash = {
        "Square 1" => 1,
        "Square 2" => 4,
        "Square 3" => 9,
        "Square 4" => 16
    }

Which is not too bad to do in one statement, albeit a complicated one. With
this library, it's pretty easy to break up complicated loops and nested loops
and other operations into a logical sequence of small individual operations.

Map, Filter, and Fold are common operations that most programmers should be
familiar with under one name or another. The to_array and to_hash convert
between the two aggregate types. There are a number of other operations in the
Query library as well: Count, Any, Single, First, First_or_Default, Take and
Skip. There are a few other operations I might like to add as well, in future
revisions, but this is a good start for now.

If you're used to System.Linq, this should be very familiar and maybe even
useful to you. I'd be lying if I said that this infrastructure didn't come
with some additional overhead compared to writing out the loops yourself, but
the tradeoff in terms of simplicity and readability will be worth it for many
people.
