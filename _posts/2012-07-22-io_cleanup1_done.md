---
layout: post
categories: [Parrot, IO]
title: io_cleanup1 Done?
---

This morning I made a few last commits on my `whiteknight/io_cleanup1` branch,
and I'm cautiously optimistic that the branch is now ready to merge. The last
remaining issue, which has taken the last few days to resolve, has been fixing
readine semantics to match some old behavior.

A few days ago I wrote a post about
[how complicated readline is](/2012/06/13/io_readline.html). At the time, I
thought I had the whole issue under control. But then Moritz pointed out a
problem with a particular feature unique to Socket that was missing in the new
branch.

In master, you could pass in a custom delimiter sequence as a string to the
`.readline()` method. Rakudo was using this feature like this:

    str = s.readline("\r\n")

Of course, as I've pointed out in the post about readline and elsewhere, there
was no consistency between the three major builtin types: FileHandle, Socket and
StringHandle. The closest thing we could do with FileHandle is this:

    f.record_separator("\n");
    str = f.readline();

Notice two big differences between FileHandle and Socket here: First, FileHandle
has a separate `record_separator` method that must be called separately, and the
record separator is stored as state on the FileHandle between `.readline()`
calls. Second, FileHandle's record separator sequence may only be a single
character. Internally, it's stored as an `INTVAL` for a single codepoint instead
of as a `STRING*`, even though the `.record_separator()` method takes a
`STRING*` argument (and extracts the first codepoint from it).

Initially in the `io_cleanup1` branch I used the FileHandle semantics to unify
the code because I wasn't aware that Socket didn't have the same restrictions
that FileHandle did, even if the interface was a little bit different. I also
didn't think that the Socket version would be so much more flexible despite
the much smaller size of the code to implement it. In short, I really just
didn't look at it closely enough and assumed the two were more similar than
they actually were. Why would I ever assume that this subsystem ever had
"consistency" as a driving design motivation?

So I rewrote readline. From scratch.

The new system follows the more flexible Socket semantics for all types. Now
you can use almost any arbitrary string as the record separator for
`.readline()` on FileHandle, StringHandle and Socket. In the
`whiteknight/io_cleanup1` branch, as of this morning, you can now do this:

    var f = new 'FileHandle';
    f.open('foo.txt', 'r');
    f.record_separator("TEST");
    string s = f.readline();

...And you can also do this, which is functionally equivalent:

    var f = new 'FileHandle';
    f.open('foo.txt', 'r');
    string s = f.readline("TEST");

The same two code snippets should work the same for all built-in handle types.
For all types, if you don't specify a record separator by either method, it
defaults to "\n".

Above I mentioned that almost any arbitrary string should work. I use the word
"almost" because there are some restrictions. First and foremost, the delimiter
string cannot be larger than half the size of the buffer. Since buffers are
sized in bytes, this is a byte-length restriction, not a character-length
restriction. In practice we know that delimiters are typically things like
"\n", "\r\n", ",", etc. So if the buffer is a few kilobytes this isn't a
meaningful limitation. Also, the delimiter must be the same encoding as the
handle uses, or it must be able to convert to that encoding. So if your handle
uses `ascii`, but you pass in a delimiter which is `utf16`, you may see
some exceptions raised.

I think that the work on this branch, save for a few small tweaks, is done.
I've done some testing myself and have asked for help to get it tested by a
wider audience. Hopefully we can get this branch merged this month, if no other
problems are found.

