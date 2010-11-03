---
layout: post
title: Blogger2Jekyll
categories: [Personal]
---

As I mentioned in my
[blog post yesterday](http://whiteknight.github.com/2010/11/02/blog_migration.html)
I had several requirements before moving my blog to a new place, but one
stood out in my mind as arguably the most important and the most
technically challenging: I had to be able to migrate all the content from my
current blog to the new system, in a more-or-less faithful manner. This
includes all blog comments, which I value very highly even if I don't have a
gigantic amount of them. I absolutely wanted to be able to bring both the
contents of my posts and the contents of all associated comments with me. I
didn't need to upconvert to new features like fancy syntax highlighting or
anything (not that anybody would even have a PIR syntax highlighter anyway),
and I did not need to convert the posts from HTML to any other kind of markup,
but I did want to bring everything of value with me.

Astute observers will notice that I *did not* bring every post from Blogger.
I went through the list and removed posts which I felt weren't relevant. All
posts are, and will remain, available for reading on Blogger (if anybody
cares). What's important is that I could have kept all those posts if I wanted
them.

Blogger provides an export format to allow you to get your raw blog data off
it's servers. The export file format is extremely dense and poorly-designed
XML, but it's better than nothing. I did a google search looking for ways to
convert the Blogger export dump file to a form more amenable to Jekyll, but
surprisingly I couldn't find anything. I figured that most of the people
moving to Jekyll would be programmers or otherwise tech-saavy people. After
all, Jekyll requires you to write your own markup in your own editor, and is
typically integrated with Git for it's version control and a handful of other
programmer tools (like Pygments) that programmers would need. If any group of
people can solve a simple data conversion task, I figured it would be Jekyll
users. There were a few solutions around. Most of them were short scripts,
one-off tools and the like. All of them had severe limitations and were
woefully incomplete for my purposes. So, myself being a programmer, I decided
to sit down and solve the problem the right way.

[Blogger2Jekyll](http://github.com/Whiteknight/blogger2jekyll) is a new
conversion tool, written by myself in C#, used to convert blogs from Blogger's
XML dump format into a format usable with Jekyll. It currently does almost
everything I wanted it to do and more, and I am still working to get it into a
shape where I am comfortable running my entire blog through it to get results
that satisfy my quality requirements. Here's a basic rundown of its feature
set:

 * The contents of all posts are converted to proper Jekyll .html files with
proper YAML front-matter
 * Unpublished posts are saved to a separate folder for drafts
 * All categories for each post are maintained in the final output
 * All comments associated with each post are included in the output text in
a section marked "Comments". Each comment maintains information about its
author.
 * All internal hyperlinks (links from one post to another in the same blog),
including links to posts, links to individual comments, and links to
categories are updated to point to the new locations.

The final feature in the list, updating all the old hyperlinks to the new
locations was the hardest problem to solve in the technical sense, and was
the last feature completed. The algorithm is pretty complex comparatively,
but it appears to have worked with 100% accuracy in my blog posts. I start
by extracting all internal hyperlinks from each post. For each, I assemble a
list of candidate posts by looking at the year and month information from the
old blogger hyperlink format. I loop over the list of candidates calculating
the [Levenshtein distance](http://en.wikipedia.org/wiki/Levenshtein_distance)
for each candidate. The "winner" of this little competition is most likely to
be the target link. Just in case there are problems, I output information
about all link mappings to a log file, including the name of the file where
the link is located in case you need to go in and make a manual
adjustment.

I will probably never have an opportunity to use Blogger2Jekyll again, since
I don't have any other blogs on Blogger that I intend to migrate and I
probably won't create any new blogs there in the future. That said, I don't
intend to actively develop Blogger2Jekyll, add new features, or anything like
that to it of my own accord. I'll happily accept patches if anybody else wants
to contribute to it, and if there are reasonable requests for new features, or
reports of bugs that affect basic operation I can probably be motivated to
work on them. The reality is though: Blogger2Jekyll worked well for me and it
is in a decent state right now that should be usable for other people.

