---
layout: post
categories: [Parrot, Rosella, Xml, Net]
title: HTML with Rosella Xml and Net
---

HTML is a derivative of SGML, just like XML is. Sure, they look pretty much the
same for the most part, but there are a few key differences that prevent HTML
from being parsed exactly like XML. Part of the reason why I like XHTML so much
is that it's more usable with more parsers, including many of simpler and
full-featured XML parsers. Simplicity in parsing was one of the original
motivations of the XML design, at least in comparison to a full SGML parser or
even something like a full HTML parser.

But that's all besides the point.

I've been in something of a backyard gardening kick lately. We bought our
house only a few short months ago, and are only half way through the first
summer growing season in my modest little garden. My plans for next year are
much more expansive. I've finally talked my wife into letting me buy some
cherry trees to plant. She was also pretty willing to get a few grape vines
planted (especially when I sketched out the beautiful wooden arbor they would
be growing on). She put her foot down when I started
talking about blueberries, apples and pears, however. And another garden bed
or two for more vegetables. For some reason she's convinced that we need some
measure of open space in our little plot so the kid has somewhere to run and
play. Some people have weird priorities.

This is all sort of besides the point too.

Getting the things I need for all this gardening work I've talked myself into
is not cheap. Cherry tree seeds actually *do grow on trees* so that's not a
big deal, but other things like fertilizers, soil amendments, tools,
materials for building a grape trellis and raised garden beds, not to mention
a longer hose to reach all the new things that are going to require regular
watering all cost money. And maybe a sprinler, like one of those fancy ones on
an electronic timer. I can avoid some of that cost by getting things used
and at discount on sites like Craigslist. So I've been going there.
Every day.

And it's tedious. I have to sort through hundreds of listings for things I
don't want, in categories that seem far too course. Sometimes, because things
often get incorrectly categorized, I have to look in other related categories
too, sorting through things that are even less relevant on average to try and
find the occasional gem. This is all on top of the hardware-related problems I
have being unable to use the trackpad on my laptop so web navigation on sites
without keyboard shortcuts is an extreme pain. I start to think to myself: I can
do better, I'm a programmer! For some values of "better" and "programmer".

Enter Rosella. Now with Parrot, Winxed and Rosella I can use the Net library
to fetch the text of the HTML code of the page. After some hacking in the
last few days, I can parse that code with my Xml library (set in a new lenient
mode) and start to work with it in a meaningful way:

    function main[main]() {
        var rosella = load_packfile("rosella/core.pbc");
        Rosella.initialize_rosella("xml", "net", "string");

        var ua = new Rosella.Net.UserAgent.SimpleHttp();
        var response = ua.get("http://philadelphia.craigslist.org/w4m/");
        var doc = Rosella.Xml.read_string(response.content, false);

        doc.get_document_root()
            .get_children_named("body")
            .get_children_named("blockquote")
            .get_children_named("p", "row":[named("class")])
            .map(function(node) {
                return {
                    "title": node.first_child("a").get_inner_xml(),
                    "link":  node.first_child("a").attributes["href"],
                    "price": node.first_child("span", "itempp":[named("class")]).get_inner_xml(),
                    "has_pic": !Rosella.String.null_or_empty(
                        node.first_child("span", "itempx":[named("class")]).get_inner_xml()
                    )
                };
            })
            .filter(function(obj) {
                return indexof(obj["title"], "compost") >= 0;
            })
            .map(function(obj) {
                return Rosella.String.format_obj("<a href='{link}'>{title} for {price}</a>", obj);
            })
            .foreach(function(string s) { say(s); });
    }

That second argument to `Rosella.Xml.read_string` tells the parser to go into
"non-strict" mode, which is basically my attempt to fudge the XML parsing rules
to allow for the SGML nonsense in HTML. Without that, the parser will blow up
pretty early in the parse because of unbalanced tags. The XML parser by
default does not handle tags which are not balanced and which do not have the
trailing slash to indicate a standalone tag, and the Craigslist source is
filled with those kinds of things.

All I need to do is set this scraper up on a timer, and have it send me
results somehow. If I set up a small server with mod_parrot and some kind of
tool for generating RSS feeds, I could have this output neatly delivered to
me on a regular basis. Considering that mod_parrot is moving along so smoothly
and RSS is just another XML format, I think this is a pretty reasonable idea.

So, I started working on that. As of last night, I've sketched out two small
libraries, one for RSS feeds and one for the competing standard, Atom. These
libraries are thin wrappers around the XML library to deal with the specifics
of RSS and Atom. Here's an example of consuming an RSS feed:

    var rss = Rosella.Rss.read_url("http://www.parrot.org/rss.xml");
    rss
        .channels()
        .first()
        .items()
        .foreach(function(i) {
            say(Rosella.String.format_obj("{title} (by {creator}) : {description}", i));
        });

You can do almost exactly the same thing with an Atom feed too, if you've got
one of those instead. Right now RSS and Atom are implemented in two separate
libraries, but I may combine them together for simplicity and to avoid
unnecessary code duplication.

I'm working on an interface to write and publish feeds as well, though that's
not quite ready yet. You can bet that when I've got that working, I'll be
setting up a copy of mod_parrot to use it with.

I've been sort of kicking around the idea of a specialized HTML parsing library,
which would more or less be an SGML parser with some schema information. I'm not
sure I want to get into that hassle because HTML is a pretty messy thing and it
will take a huge amount of effort to get something that works most of the time.
But, if you're willing to put up with a little bit of oddity, the Xml library
works well enough for many cases.


