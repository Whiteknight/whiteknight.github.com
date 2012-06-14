---
layout: post
categories: [Parrot, IO]
title: Reading a Line of Text
---

In terms of usage, there aren't too many IO-related features in Parrot's
user interface than the `readline` method. It does exactly what you tell it
to do: read a line of text from the given file and return that line of text
as a Parrot string. Easy.

Tonight I was looking at some of the old code to get an idea about expected
semantics for some tests that need fixing. Let's look at some code:

    .sub read_a_line
        .param string type
        $P0 = new [type]
        $S0 = $P0.'readline'()
        .return($S0)
    .end

    .sub test_readline
        $S0 = 'read_a_line'('FileHandle')
        say $S0
        $S0 = 'read_a_line'('Socket')
        say $S0
        $S0 = 'read_a_line'('StringHandle')
        say $S0
    .end

The valid types for this are, as usual, `"FileHandle"`, `"Socket"` and
`"StringHandle"`. Notice that we're reading a line from the object of the
given type before we've opened, connected or initialized. Pretend, in order
to save myself some typing, that I've set up exception handlers and the like
above. So, what happens?

* For `FileHandle` we throw an exception. You can't read from a closed handle.
* For `StringHandle`, we throw an exception for the same reason.
* For `Socket` we return null because...whatever. (in the test suite we test
  that when converted to a floating-point number, that it's 0.0. Again,
  whatever).

So that's a little bit weird that socket does something different from the
other two, but fundamentally it's a pretty different type so I suppose some
differences can be allowed.

Now, let's try something slightly different:

    .sub read_a_line
        .param string type
        $P0 = new [type]
        $P0.'open'("foo.txt", "r")
        $P0.'print'("This is \n test text")
        $P0.'close'()
        $S0 = $P0.'readline'()
        .return($S0)
    .end

With this example we can only operate on `FileHandle` and `StringHandle`
because `Socket` doesn't have an `.open()` method like those two do. What does
this do for those two types?

* For `FileHandle` we throw the same exception, you still can't read from a
  closed handle.
* For `StringHandle` you can *read like normal* without any indication that
  the handle is closed!

So that's weird to say the least that StringHandle has two different
behaviors. `Socket` has yet another problem, in a slightly different way.
The method `Socket.readline()` returns null when not open, but if you pass
a `Socket` to the `Parrot_io_readline` method, it always throws an exception
because apparently readline on a `Socket` isn't supported! And because
readline on a `Socket` uses a completely different code path from `FileHandle`
the two types use completely different buffering mechanisms with subtly
different semantics (`StringHandle`, because it uses the in-memory string
buffer, does it in a third way).

To recap: What is conceptually a simple operation, read in some text until we
find a delimiter, is done in three completely different ways by three
different types, each with different error-handling semantics depending on
both history, state, and the interface used. If anybody was wondering why I
wanted to rewrite this subsystem, here's part of the reason.

Actually, I kind of lied. It's really not a simple operation which is all the
more reason we should share common code. It's a clear case of an algorithm
where the hard parts should be encapsulated inside a clean interface so that
different types can avoid needing to reimplement it over and over again (with
differences, bugs and complications). That's the way it really should be,
but some of the complications in the code are a little hard to live with.
Here's the general algorithm for readline on a `FileHandle`, as it's
implemented in Parrot master:

1. The filehandle requires a buffer for this, so create (and fill) a buffer
   if one isn't configured.
2. Create a new, empty STRING header.
3. Treating the buffer like an encoded STRING, scan the buffer looking for
   the end of the delimiter or the end of the buffer, whichever comes first.
4. Allocate/reallocate enough space in the STRING header to hold all the
   data we've found in the buffer.
5. Append all the characters we've found to the STRING.
6. If we've found the delimiter, we're done. Return it to the user.
7. Otherwise, check if we are at the end of file for the input. If so, go to
   #8. If not end of file, go to #9.
8. Check that the last codepoint is complete and has all its bytes. If so,
   return the STRING to the user. If not, throw an exception about a
   malformed string.
9. Check that the last codepoint is complete and has all its bytes. If so,
   go to #10. Otherwise, go to #11.
10. Refill the buffer and go to #3.
11. Determine how many more bytes we need to read to complete the last
    codepoint.
12. Refill the buffer, and check that we have at least that many bytes
    available to read. If so, go to #13. Otherwise, throw an exception about
    a malformed string input.
13. Read in the necessary number of bytes (1, 2 or 3 at most) from the buffer
    and go to #3.

If you're reading an ASCII or fixed8 string the logic obviously collapses
down to something a little bit more manageable. Also, this same logic, almost
line for line, is repeated in the routine to read a given number of characters
from the handle, where characters in a non-fixed-width encoding (like utf8)
may need multiple reads to get if we don't get all the bytes for the
character into the buffer in a single go. Notice that the versions provided
by StringHandle and Socket are both much more simple and not safe for
multi-byte encodings like `utf8` or `utf16`.

In my `io_cleanup1` branch, the logic has been simplified substantially, and
a single codepath is now used for all three of the major types:

1. Make sure the handle has a read buffer set up and filled.
2. Create a new, empty STRING header.
3. Ask the buffer to find the given end-of-line character. The buffer will
   return a number of bytes to read in order to get a whole number of
   codepoints, and a flag that says whether we've found the delimiter or not.
4. Append those bytes to the string header.
5. If the delimiter is found or if we are at EOF, return the string.
6. Fill the bufffer and go to #3.

By simply coding the buffer logic to refuse to return incomplete codepoints in
response to a STRING read request, the whole algorithm becomes hugely
simplified. The readline routine in master takes up 185 lines of C code. In my
new branch, the same routine takes up only 47 lines. Of course, this isn't
comparing apples to apples, because I did break up some of the repeated logic
into helper routines, and the buffers in my system are obviously a little bit
smarter about STRINGs and codepoints, but that's not exactly the point. The
real point is that three large, complicated, hard-to-read functions in master
are now a single, much smaller, easier-to-read routine that relies on clear
abstraction boundaries to do a difficult job in a much more conceptually
simple way.

I've also updated the STRING read routine (now called `Parrot_io_read_s`)
to use a similar algorithm and actually share some of the new helper methods.
That sharing itself also helps to decrease total lines of code has has other
benefits as well.

Notice that there is one small change in these two algorithms, which may or
may not need to be worked around if it causes problems. Notice that we don't
read out of the buffer an incomplete codepoint. If we have an incomplete one
at the end of the file, the first algorithm will read it in and throw an
exception about a malformed string. The second algorithm will ignore those
final bytes and successfully return all the rest of the valid-looking data
from the buffer instead. In the first algorithm, it then becomes impossible
to read the partial data out and make a best effort, while in the second
algorithm you can easily get to the data, even if the last codepoint is
corrupted and cannot be read. I'd really love to hear what people think about
this change, and whether it's worth keeping or needs to change. I suspect it
is better this way but only the users can really say for sure.
