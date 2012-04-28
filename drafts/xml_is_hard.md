---
layout: post
categories: [Parrot, Rosella, Xml]
title: XML Is Hard
---

Last week I promoted the Parse and Json libraries in Rosella to stable status.
For both those libraries I wrapped up a few outstanding TODO issues, wrote up
some website documentation and added a bunch of unit tests. I figured I would
do the same thing for the XML library too. After all, I had done the hard part,
the first 90% of the library was the recursive descent parser which I had most
of.

So today I got to work on that library, trying to put together the last few
bits so I could make the library stable. Like I said, I had about 90% of it
done earlier. I spent the time today doing another 90%. I figure I only have
about 90% left to go before I have a "real" XML library. Somewhere a
mathematician is reading this post and inventing new curse words.

It turns out that XML is hard.

Anybody can put together a little parser for XML-like tag syntax with
attributes, text, and nested tags. It's once you start getting into DTD
declarations and schema validation that things get messy. Honestly, I don't
think I can seriously call Rosella's XML library "complete" without those
things. Or, not without most of them. I can probably get away with only the
first 90% or so.

So, what can Rosella's Xml library do today? Here is a sample of XML text that
I can parse without problems:

    <?xml version="1.0"?>
    <!DOCTYPE foo [
        <!ELEMENT foo (bar, baz)>
        <!ELEMENT bar ANY>
        <!ELEMENT baz (fie)>
        <!ELEMENT fie EMPTY>
        <!ATTLIST fie
                    lol CDATA #REQUIRED
                    wat CDATA #IMPLIED
                    sux CDATA #FIXED "hello!">
    ]>
    <foo>
        <bar/>
        <baz>
            <fie lol="laughing out loud" wat="you talkin bout?" sux="hello!"/>
        </baz>
    </foo>

Or, if I want, I can jam all that schema nonsense into a separate file, and load
it separately:

    <!DOCTYPE foo SYSTEM "foo.dtd">

I haven't integrated Rosella Net yet, to allow loading schemas from a URL. In
code, I can do a few things:

    var dx = new Rosella.Xml.Document();
    dx.read_from_file("foo.xml");
    dx.validate();
    if (!dx.is_valid()) {
        for (string err in dx.errors)
            say(err);
    }

    var dtd = new Rosella.Xml.DtdDocument();
    dtd.read_from_file("foo.dtd");
    var errors = dtd.validate_xml(dx);
    if (elements(errors) > 0) {
        for (string err in errors)
            say(err);
    }

That example shows us loading an XML document from a file and validating it with
it's built-in rules from the `!DOCTYPE` header. The second part shows us loading
a separate DTD definition from a standalone file, and using that to validate the
XML document too. In both cases, the validator runs through the document object
and returns a whole list of error messages, not just a simple yes/no flag.

So what is left to do? Well, for starters there's a bunch of syntax in the
`!ELEMENT` tag that I don't quite handle yet, such as quantifiers and
alternations:

    <!ELEMENT foo (bar*, (baz|bar), fie?)>

Parsing all that in a way that doesn't suck is not something I'm looking forward
to doing.

Then in attribute lists, there's some syntax I don't deal with, such as
enumerated values again:

    <!ATTLIST foo bar (yes|no)>

The validator I've implemented is pretty naive so far, and isn't set up to do
quantifiers anyway. That's all going to take a while to do. We're doing some
basic validation now, but nowhere near as much as we would expect from a full
implementation.

My Json library is about 1300 lines of winxed code long, including whitespace.
My Xml library is about 2400 lines of code long and still growing. Json is
pretty easy (by design!), but XML is very hard. I'm not going to push the Xml
library to become stable any time soon, there's a hell of a lot of work left on
it and I'm not going to rush anything.
