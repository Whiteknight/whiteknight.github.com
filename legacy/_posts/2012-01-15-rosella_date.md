---
layout: post
categories: [Parrot, Rosella, Date]
title: Rosella Date
---

The Advent calendar idea ended up terribly. Let us never speak of it again.
The holiday season was particularly busy this year with a higher-than-average
load of family- friend- and work-related activities. Combine that with an
unexpected (and absolutely unappreciated) invasion of a particularly
unpleasant and long-lasting stomach bug, and you have something of a perfect
storm. I won't go into the details any further on this particular blog, but I
will mention in passing that I've become extremely suspicious about the basic
hygene habits of the other little turdbags that my son goes to daycare with.

To start 2012 off, and to get back into the swing of coding-for-pleasure,
yesterday I went through and got the new Rosella Date library ready for prime
time. The library is imperfect and incomplete, but those are things that can be
fixed using the patented Andrew Style (tm) of meandering iterative development.
For now, however, the library does seem to work well enough for basic tasks and
it's already proven to be a valuable tool for some tasks. Here I'm going to
introduce the new library and talk about some of the things I changed in
Rosella to support it and some of the ways I've already integrated it into the
rest of the collection.

### String Formatters

When you use `sprintf` to print out an integer (for instance) you can use some
basic modifiers to control how the value is printed. `%d` prints out the basic
version, but you can specify field width, padding, alignment and a few other
details by using a format specifier like `%-02d`. Or, if you want to print the
same value out as hex instead of base-10, you can use `%x` or `%X`, and use
modifiers with those as well.

The problem with something more complex like a date/time representation is that
sprintf is not able to handle them natively and some kind of mapping is needed.

Rosella has added a new `StringFormatter` type to the Core library to help with
this problem. A StringFormatter is a type that takes an object and a format
string and outputs a new string according to the two. The default
StringFormatter uses sprintf internally, but other formatters may use different
mechanisms and syntaxes.

    var sf = new Rosella.StringFormatter();
    string s = sf.format(my_obj, "This is %s");

As an aside, this does demonstrate the fact that our `get_string` vtable really
is insufficient for a lot of purposes. I suggest that `get_string` should take
a parameter for a format string. We could easily incorporate that into the
default sprintf implementation like this:

    $S0 = sprintf "%{foobar}p", $P0

In that invocation, the `get_string` vtable on the first parameter would be
called with the string argument "foobar". A normal invocation like this:

    $S0 = sprintf "%p", $P0

...would call `get_string` with a null format and the behavior could be whatever
the default string representation for that type is. By overriding `get_string`
in your types to take different formats or to respond to common formats
differently, you could have pretty detailed control over stringification at all
levels.

### Date Library

The new Date library provides a Date type for working with dates and times. You
can create one in three ways:

    var d = new Rosella.Date(t);
    var d = new Rosella.Date(year, month, day);
    var d = new Rosella.Date(year, month, day, hour, minute, second);

The first uses a system-specific time integer value to represent a time since
the system epoch. This is the kind of value you get from the `time` opcode, or
from `stat` calls on filesystem objects, for instance. In the third option,
hours are specified on a 24-hour clock.

There are also a few functions you can call to get particular dates:

    var d = Rosella.Date.now();
    var n = Rosella.Date.min();
    var x = Rosella.Date.max();

The first value should be the current date/time (as gotten from Parrot's
`time` opcode). The second value is a minimum date object which corresponds to
the minimum possible date value to display and is guaranteed to be evaluated
as less than any other date. The third one similarly is a maximum date value
which corresponds to the maximum possible date and is guaranteed to always be
compared greater than any other date.

Dates are immutable. Once you create them, they cannot be modified in-place.
Instead, several operations are provided to perform operations and return new
Date objects with the results. Here are some examples:

    var d = Rosella.Date.now();
    var e = d.add_seconds(20);
    e = d.add_hours(15);
    e = d.add_months(24);
    e = d.add_years(1000);

In each case, the variable `e` becomes a new date value and `d` is left
unmodified. Two other methods let you pick out just the date components or just
the time components:

    var n = Rosella.Date.now();
    var d = n.date();
    var t = n.time();

Finally, you can use a new DateFormatter type to format the date value into a
proper stringification:

    var d = Rosella.Date.now();
    string s = d.format_string("yyyy - MM - dd and the time is: hh:mm:ss");

At the moment the formatter is dirt simple and only supports a few formatting
codes such as `yyyy`, `MM`, `dd`, `hh`, `mm`, and `ss`. I will be making it much
more useful in the coming days, if I can settle on an algorithm which doesn't
completely stink.

### FileSystem

The FileSystem library now returns Date objects from certain file-time methods:

    var f = new Rosella.FileSystem.File("t/harness");
    var ct = f.change_time();
    var at = f.access_time();
    var mt = f.modify_time();

In each case, the value returned from the `stat` call on the file is used to
create a new Date object.

### Query

Dates are completely comparible, so you can sort them and work with them like
other values in an iterable:

    var a = [
        Rosella.Date.now(),
        Rosella.Date.max(),
        Rosella.Date.min()
    ];
    Rosella.Query.iterable(a)
        .sort()
        .foreach(function(d) { say(d); });

This toy example sorts the three Date values, with the min first, then now, then
max, and prints them out to the console. The stringified versions of the special
min/max dates aren't particularly instructive, but they do get the point across.

So that's the new Date library. It does need more functionality and definitely
needs more tests, but it is working pretty well for me now and has already
proven itself useful for a number of purposes. Expect to see more of it in 2012.
