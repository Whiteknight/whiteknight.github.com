---
layout: post
categories: [Parrot, Rosella, Xml, Net]
title: HTML with Rosella Xml and Net
---

HTML in general is not known to be a well-formed derivative of XML. Sure, it
uses the same kinds of tag markup, but some of the legacy issues involved in
HTML prevent it from being read with a strict XML parser. The fact that
browsers continue to render old sites with broken markup means that there's
no real impetus for modern popular sites to fix their problems. Calling these
broken things "standards" and letting people validate against them doesn't
help the problem either.

But that's all besides the point.

I've been in something of a backyard gardening kick lately. We bought our
house only a few short months ago, and are only half way through the first
summer growing season in my modest little garden. My plans for next year are
much more expansive. I've finally talked my wife into letting me buy some
cherry trees to plant. She was also pretty willing to get a few grape vines
planted (especially when I sketched out the beautiful wooden arbor they would
be wrapped around on the back porch). She put her foot down when I started
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
watering all cost money. I can avoid some of that cost by getting things used
and at discount on sites like Craigslist. So I've been going there.
Every day.

And it's tedious. I have to sort through hundreds of listings for things I
don't want, in categories that seem far too course. Sometimes, because things
often get incorrectly categorized, I have to look in other related categories
too, sorting through things that are even less relevant on average to try and
find the occasional gem. I start to think to myself: I can do better, I'm a
programmer!

Enter Rosella. Now with Parrot, Winxed and Rosella I can use the Net library
to fetch the text of the HTML code of the page. After some hacking in the
last few days, I can parse that code with my Xml library too, and start to
work with it in a meaningful way:

    function main[main]() {
        var rosella = load_packfile("rosella/core.pbc");
        Rosella.initialize_rosella("xml", "net", "string");

        var ua = new Rosella.Net.UserAgent.SimpleHttp();
        var response = ua.get("http://philadelphia.craigslist.org/gra/");
        var doc = Rosella.Xml.read_string(response.content, false);
        var post_links = doc.get_root_element()
            .get_children_named("body")
            .get_children_named("blockquote")
            .get_children_named("p", "row":[named("class")])
            .get_children_named("a");
        for (var post in post_links) {
            string title = post.get_inner_xml();
            title = Rosella.String.trim(title);
            title = Rosella.String.to_lower(title);
            if (indexof(title, "compost") >= 0) {
                say(title);
        }
    }

That second argument to `Rosella.Xml.read_string` tells the parser to go into
"non-strict" mode. Without that, the parser will blow up pretty early in the
parse because of unbalanced tags and all sorts of other garbage. The parser by
default does not handle tags which are not balanced and which do not have the
trailing slash to indicate a standalone tag, and the Craigslist source is
filled with those kinds of things.

I'm working to integrate the Rosella Query and String libraries as well, so we
can use some higher-order functions and better string-handling methods. Here's
an example of something I would like to be able to do in the near future:

    doc.get_root_element()
        .get_children_named("body")
        .get_children_named("blockquote")
        .get_children_named("p", "row":[named("class")])
        .map(function(node) {
            return {
                "title": node.first_child("a").get_inner_text().to_lower(),
                "link":  node.first_child("a").attributes["href"],
                "price": node.first_child("span", "itempp":[named("class")]).get_inner_text(),
                "has_pick": !Rosella.String.null_or_empty(
                    node.first_child("span", "itempx":[named("class")].get_inner_text()
                )
            };
        })
        .filter(function(obj) {
            return indexof(obj["title"], "compost") >= 0;
        })
        .map(function(obj) {
            return Rosella.String.format_obj("<a href='{link}'>{title} for {price}</a>', obj);
        })
        .foreach(function(string s) { say s; });

Then all I need to do is set this scraper up on a timer, and have it send me
results somehow. If I set up a small server with mod_parrot and some kind of
tool for generating RSS feeds, I could have this output neatly delivered to
me on a regular basis. Considering that mod_parrot is moving along so smoothly
and RSS is just another XML format, I think I can make this happen.

Part of me wants to create a new HTML library, based on the existing XML
parser. But HTML has so many different formats and rules, and all sorts of
legacy garbage to deal with, that I just don't think it's worth my time. The
XML parser can do just fine with it so far, and despite a few hiccups it works
the way I need it to. Of course, if Jaesop ever takes off, I'd be able to
integrate that into an HTML library for running JavaScript directly in the
DOM, but that's another issue entirely for consideration far in the future.
