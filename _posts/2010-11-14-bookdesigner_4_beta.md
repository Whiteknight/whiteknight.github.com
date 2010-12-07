---
layout: post
categories: [MediaWiki, EmbedVideo]
title: BookDesigner Version 4 Beta Testing
---

I've been working on the [BookDesigner][bookdesigner] tool for a very long
time. It started several years ago when I was still an active
[Wikibookian][meatwikibooks] and I was looking to
create a tool, both for myself and for others, to help with the process of
creating books. I had taken a lot of feedback from other users, especially new
users, and I found through everybody I talked to that creating new books was
difficult. Creating them following the best practices as developed by the
Wikibooks community was even harder. I created my tool to help simplify those
tasks and automate some of the tedious and nuanced parts of it to help speed
along new users who had content in their heads that they wanted to share, but
didn't want to waste their time putting together the structure of a complete
book.

[meatwikibooks]: http://en.wikibooks.org/wiki/User:Whiteknight

The first version of the tool was a simple form with a textbox. The user
entered in a script using a custom toy language I invented for the purpose. It
was little more than an outlining tool, where asterisks and other symbols were
used to denote the shape of the book and the contents of individual pages. I
used this tool for my own needs for a while, but the feedback I got from
others was that it was arcane and difficult to use. With that in mind, I set
off on the biggest JavaScript project I had ever undertaken up till that point
and tried to create a user-friendly version of it instead.

The second version, which I called my "Visual Book Designer" looked very
similar in many respects to the tool I have now: It used a graphical
representation of an outline instead of a text-based one and was much easier
to use, not to menton much better to look at. This tool was well-received and
I used it to help create several books for myself and others while at
Wikibooks. What it did not do well was actually automate the process of
creating the individual pages. I had developed some custom AJAX routines to
automate the task, but it never worked nearly as well as I would have liked.

Eventually I was asked to help do some [contracting work][wittie], and decided
that this was the time to convert the tool to a proper extension instead of an
assortment of JavaScript files stored in the wiki itself. Instead of a long
series of hackish AJAX calls, I wrote all the backend in PHP and could create
pages directly in a single server request. This was version 3, and while it
had a [few bugs and few annoyances][issues], it was used on several occasions
to create particularly large books.

Today, I've added most of the big new features and am starting serious beta
testing for version 4 of the tool. The interface still has not
changed a whole hell of a lot since version 2, but the internals have improved
pretty dramatically. Here are a list of the changes since Version 3:

 * The data transmission format uses XML instead of a hackish custom markup
   than I was using previously.
 * The extension, for the most part, is i18n compatible and most user-facing
   text can be translated into other languages
 * Internally, the architecture is much improved, using a series of objects
   instead of being one big class of static methods.
 * There is now a confirmation page where you can view the list of all pages
   to be created and their text. From the confirmation page you can select
   which pages are created.
 * Several bits of text, including default text for generated templates, can
   be edited using system message pages for site-specific defaults
 * The text of the outline can be saved to a page on a wiki or downloaded
   as a file to the user's computer. These saved outlines can be loaded back
   into the tool at any time to continue work on the outline.

All told it's been a large amount of work but well worth it. I sincerely hope
that this tool will be useful for other people and that it will help many
new authors to create great books on MediaWiki wikis.

There are a few things I need to tweak still, and I need to get testing done
on a few versions of MediaWiki (I've only tested on 1.16.0 so far). When all
that is done, I would like to cut the official release and submit a version of
it to the MediaWiki extensions repository.

[bookdesigner]: http://github.com/Whiteknight/mediawiki-bookdesigner
[issues]: https://github.com/Whiteknight/mediawiki-bookdesigner/issues
[wittie]: http://wittieproject.org
