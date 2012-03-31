---
layout: post
categories: [Parrot, Rosella, Xml, Json]
title: Rosella Json and Xml
---

In a [previous post][rosella_net_post] I wrote about how I didn't have XML or
JSON parsing libraries in Rosella and didn't have any real plans to build them
either. Well, I kind of lied. As of this morning, Rosella has an experimental
prototype of an XML parsing library and a JSON library.

On the XML side, you can do something like this:

    var d = new Rosella.Xml.Document();
    d.read_from_string("<foo><bar value='hello!'/></foo>");
    say(d.get_root_element().children[0].attributes["value"]);
    d.write_to_file("test.xml");

For JSON you can do similar things:

    var obj = Rosella.Json.parse("{ 'foo':3.14, 'bar': [ null, []] }");
    string json = Rosella.Json.to_json(obj);

I was looking at the ByteBuffer and StringIterator types, which both allow
relatively quick access to individual characters as integer values. Originally
I didn't want to make an XML or JSON parsing library because I figured that
string/substring operations to break the string up would be far too slow.
However, if you iterate the strings as integers and make ample use of Winxed's
compile-time constants and code inlining abilities, the performance can suddenly
become comparable to any other data parser in other similar languages.

Things could actually be faster still if I could assume all strings are ASCII.
However real-world data doesn't play by those rules. So, we need to go through
the added layer of abstraction in StringIterator to get characters, which might
not be fixed-width depending on encoding. Things could also be faster if
StringIterator had a fast "unget" or "roll back" operation. Instead I need
to maintain a relatively costly character stack for parsing, which is not quite
ideal.

Are these libraries standards compliant? No. Does the XML library know anything
about XSLT, XPath, SAX, or schemas? No.  Do these libraries handle all error
conditions gracefully? No. Are they well tested right now? Emphatic no.

Parrot's standard library comes with a `data_json` compiler object which can
compile JSON to an object, or serialize an object back to JSON. I haven't
benchmarked my implementation against that one, but I have many reasons to
believe that mine is significantly faster. It's still immature and probably
very error prone, but it's only existed for a day and there are many
improvements left to make.

These are basic and immature implementations of data format parsing libraries,
and excellent proofs of concept. I need to add plenty of tests for each and
then start benchmarking to make sure the bit-twiddly effort I've put in to
creating quick algorithms has paid off. Rosella has two new toys to play with,
and I hope they can do some good in the coming months.
