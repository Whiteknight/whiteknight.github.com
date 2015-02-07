---
layout: post
categories: [Parrot, IO]
title: Sockets and Encodings
---

The `io_cleanup1` branch is nearing completion, though as always the last few
details are what holds everything up. In the past few days all the
remaining tests in the parrot repo were passing. The coding standards tests,
as usual, the last to be resolved. Then I started building and testing
other things on the branch: Winxed builds and tests fine. So does Rosella.
Then I looked at NQP and Rakudo. Both built fine, but Rakudo was failing two
socket-related spectests.

That's not entirely unexpected. Even though my intention was to make this
branch as painless as possible there were still some unavoidable changes to
interfaces and semantics. There are a few places where older semantics are
surrounded by large `/* HACK! */` comments, but for the most part I've tried
to make everything sane. That's why I wasn't surprised to see Rakudo failing
a few tests. I was much more surprised that Rakudo built without any problems
the first time I tried it. I figured the test failures represented some kind
of semantic mismatch, and getting Rakudo passing again would have been as
easy as getting the old semantics returned, with a note about a future update
path.

It turns out this wasn't exactly the case. For one test it was the simple
difference in the way we read on streams with multibyte encodings. This was
expected and we can fix it to use the old behavior if that's what Rakudo
prefers. For the second failing test, it's not that there's a semantic
difference per se, but instead there is a glaring and serious bug in master
that was corrected in the new branch. Here, I'm going to explain what's going
on.

Look at this code:

    Parrot_io_recv_handle(PARROT_INTERP, ARGMOD(PMC *pmc), size_t len)
    {
        Parrot_Socket_attributes * const io = PARROT_SOCKET(pmc);

        /* This must stay ASCII to make Rakudo and UTF-8 work for now */
        STRING * res    = Parrot_str_new_noinit(interp, len);
        INTVAL received = Parrot_io_recv(interp, io->os_handle,
                                         res->strstart, len);

        res->bufused = received;
        res->strlen  = received;

        return res;
    }

This is a pared-down version of the code behind the `recv` method on Socket.
It creates a new string with the specified length pre-allocated, then passes
the buffer to the low-level `recv` C API (which has been abstracted a little
to account for platform differences).

Notice the comment there in the middle which says the string uses the ASCII
encoding, for use by Rakudo. This is what I saw, and this is the semantic
I followed in the new system: When you read from a socket by default in the
new system, the string is encoded as ASCII unless you specify differently.

Just for my own verification, I had to look at the `Parrot_str_new_noinit`
function to verify that the string was, in fact, being set to ASCII:

    Parrot_str_new_noinit(PARROT_INTERP, UINTVAL capacity)
    {
        STRING * const s = Parrot_gc_new_string_header(interp, 0);
        s->encoding = Parrot_default_encoding_ptr;

        Parrot_gc_allocate_string_storage(interp, s,
            (size_t)string_max_bytes(interp, s, capacity));

        return s;
    }

Elsewhere in the system, we have this:

    Parrot_default_encoding_ptr = Parrot_ascii_encoding_ptr;

So yes, the string returned by the Socket does indeed use the ASCII encoding
in master. And, after double-checking, the version in the `io_cleanup1` branch
was using ASCII also. However, in the new branch Rakudo's test fails because
of an exception about a lossy conversion of non-ascii data into the the
lower bit-width format. A quick check shows that both systems create an ASCII
string buffer and both systems call the same `recv` function to fill it. So
where's the problem? What the hell?

For comparison, here's the snippet of code from the new branch that reads
data into a STRING, possibly using a buffer:

    bytes_read = Parrot_io_buffer_read_b(interp, buffer, handle, vtable,
                                       s->strstart + s->bufused, byte_length);
    s->bufused += bytes_read;
    STRING_scan(interp, s);

We're reading out a number of bytes, appending them into the string's
pre-allocated storage and updating the number of bytes actually used. That's
all the same as in master. However, the last line, `STRING_scan` does not
appear in master. What is it?

`STRING_scan()` loops through the data in the string to verify that it
correctly matches the string's encoding. For instance, if the string is
encoded as ASCII, `STRING_scan` will loop through to make sure all character
values are lower than 128. If the string is UTF-16, `STRING_scan` verifies
that we have an even number of bytes and that each value is an acceptable
codepoint.

`master` doesn't do this, which means there is a bug. In master, we don't
scan the string after `recv` but before we return it to the user, which means
we can have non-ASCII data in a string marked with the ASCII encoding. The
Rakudo test puts UTF-8 data into the socket on the server side, and then reads
out a string and encodes that to UTF-8 to verify that it comes out correctly.
However in the new branch we actually check that the string is valid before
giving it out to user code, and it isn't, so we throw an exception.

Combine that with the fact that the Socket PMC has no way to change the
encoding it uses in master, which means all Sockets used in Parrot master are
potential sources of bugs.

Two nights ago I added methods to Socket to get/set the encoding to use, and
everybody's favorite Moritz created a branch for Rakudo to use it. Last night
I did some playing with default encodings. Tonight and into the weekend I'm
hoping to wrap up the last few details to get the Rakudo spectest passing like
normal again. Hopefully, if all goes well, we can start talking about a merger
within the next week or two.
