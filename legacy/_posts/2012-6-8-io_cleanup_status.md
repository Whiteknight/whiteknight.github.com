---
layout: post
categories: [Parrot, IO]
title: IO Rewrite Status
---

I was going to call this post "IO Cleanup Status", but let's face facts: This
is a complete rewrite of the entire subsystem. I haven't hardly left a single
line of code untouched. It is a full rewrite of the system hiding behind a
mostly-similar (though not quite the same) API. I didn't intend to completely
rewrite the whole subsystem when I started the branch, hence the
benign-sounding branch name. Following along with our cultural norms, I
could have called it `whiteknight/io_massacre` or something similarly
upbeat. Whatever. I've known people stuck with un-liked names for their
entire lives, so this branch can be misnamed for a few weeks.

So what is the status of this branch, exactly?

At the time of this writing the branch is mostly complete. The major
architectural work has all been done, with per-type logic separated out into
new `IO_VTABLE` structures, and buffering logic divorced from FileHandle into
a new `IO_BUFFER` structure. Now you can do things that have never been
possible before, like buffering socket input and output, or doing readline
with custom line-end characters on all handle types, and a whole bunch of
other, increasingly-obscure operations. A lot of the new capabilities are
things you didn't even know we didn't support before. Now, we do.

We aren't quite there yet, but the stage is set for some other awesome
changes in the future too, which I'll talk about in more depth when we get
there.

The current status of the branch is good. Parrot builds without any huge
amount of new warnings and with no errors on my platform. Some
platform-specific code needs to be updated for Windows, I'm sure. The one big
thing standing in the way is keeping track of file positions through
operations like `seek` and `tell`. These things are made a little bit more
difficult when you have read buffers reading ahead, because the position of
the next character to read according to the user may be far different than the
position of the file descriptor according to the operating system. Then
consider the case when you have a file opened for read and write, with
buffers in both directions. The old system had a single buffer per FileHandle
which needed to be flushed if you tried to read when the buffer was in write
mode, or you tried to write when it was in read mode. If you're switching
back and forth between reading and writing often enough, buffering actually
decreases performance when it's supposed to be a performance enhancer.

The FileHandle has an attribute to keep a pointer to the current cursor
location, but I'm not always updating it as often as I should and not always
reading it when I should. If you have a file opened for read and write, when
you write 5 characters at the current file position you need to increment
the read buffer by 5 characters also. When you go to read in 5 characters from
the current position, you either need to flush the write buffer first or you
can try to read those characters right out of the write buffer. There's
nothing complicated about it, just a lot of bookkeeping to get right and lots
of little interactions that need to be tested. It's helpful that we don't do
`seek` or `tell` on some things like Sockets, and we don't really buffer
StringHandles.

The branch is moving along well and if I can find the time to actually sit
down and work on it for a dedicate period of time I might be able to get it
closer to being done. I'm shooting for being mergable sometime after the
coming release.

