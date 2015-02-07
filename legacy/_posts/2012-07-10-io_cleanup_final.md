---
layout: post
categories: [Parrot, IO]
title: IO Cleanups Home Stretch
---

I had made a round of fixes with regards to encodings in the
`whiteknight/io_cleanup1` branch a few days ago. Rakudo hacker Moritz was
able to take a look at Rakudo's spectests and verify that more tests were
indeed passing because of it. The remaining test failures represent the
changing semantics for the `read` method and what appear to be two genuine
regressions or bugs.

Hopefully I will be able to get all these things sorted out this week before
I go away on a mini vacation next weekend. Otherwise I can't imagine this
branch gets merged before the 4.6 release this month.

A few days ago I wrote a [post about readline](/2012/06/13/io_readline.html)
and some of the intricacies involved in that, and some of the weird semantics
that I was attempting to unify. It turns out that some of these semantics are
a major cause in one of the last bugs in the branch. Let's look at some code
in master to see where the hangup is. First, `readline` on a Socket:

    METHOD readline(STRING *delimiter    :optional,
                    INTVAL has_delimiter :opt_flag) {
        INTVAL idx;
        STRING *result;
        STRING *buf;
        GET_ATTR_buf(INTERP, SELF, buf);

        if (!has_delimiter)
            delimiter = CONST_STRING(INTERP, "\n");

        if (Parrot_io_socket_is_closed(INTERP, SELF))
            RETURN(STRING * STRINGNULL);

        if (buf == STRINGNULL)
            buf = Parrot_io_reads(INTERP, SELF, CHUNK_SIZE);

        while ((idx = Parrot_str_find_index(INTERP, buf, delimiter, 0)) < 0) {
            STRING * const more = Parrot_io_reads(INTERP, SELF, CHUNK_SIZE);
            if (Parrot_str_length(INTERP, more) == 0) {
                SET_ATTR_buf(INTERP, SELF, STRINGNULL);
                RETURN(STRING *buf);
            }
            buf = Parrot_str_concat(INTERP, buf, more);
        }

        idx += Parrot_str_length(INTERP, delimiter);
        result = Parrot_str_substr(INTERP, buf, 0, idx);
        buf = Parrot_str_substr(INTERP, buf, idx, Parrot_str_length(INTERP, buf) - idx);
        SET_ATTR_buf(INTERP, SELF, buf);
        RETURN(STRING *result);
    }

We can ignore the fact that this implementation of `readline` doesn't call
`Parrot_io_readline` like every other PMC does. Or that if we did call that
function the program would throw an exception because `Parrot_io_readline`
doesn't support sockets anyway. Whatever. Moving on...

For comparison, let's look at the version from the Handle PMC (which is
inherited by FileHandle):

    METHOD readline() {
        STRING * const string_result = Parrot_io_readline(INTERP, SELF);
        RETURN(STRING *string_result);
    }

The Socket version takes a `delimiter` parameter which is a STRING. When doing
readline on a Socket, you can pass in any arbitrary string which is used as
the token for end of line. With FileHandle, you don't seem to have that.
However, you can definitely use custom delimiters with FileHandle. However,
we clearly don't take a delimiter here and we aren't passing one in as an
argument to `Parrot_io_readline` like we do in the branch. Let's see
how it's done instead. Here's a snippet from Handle PMC:

        ATTR INTVAL    record_separator;  /* Record separator (only single char supported) */

We don't need to look at any other code. This is the smoking gun.
`Socket.readline()` can take any arbitrary STRING to use as a record
separator, but `FileHandle.readline()` can only use a single codepoint, which
it doesn't take as an argument.

So that's the problem right there. When I standardized the readline mechanics
between types, I picked the FileHandle semantics. This was probably the wrong
decision, because not only could Sockets use a more general mechanism but
Rakudo relies on that behavior in its spectests. This does raise a question
about why nobody ever expected this same behavior from FileHandle, or why the
difference was not considered some kind of bug. It really goes to show how
immature our IO system has been for all these years, and how we had all just
grown accustomed to the arbitrary, inconsistent, nonsensical behaviors. It
just works for some basic usages, so nobody ever complains about it. That
time is, thankfully, coming quickly to an end.

Fixing this issue is actually going to take some serious work. Several
function signatures are going to need updating to take a STRING delimiter
instead of an INTVAL codepoint, and a major chunk of buffering logic is going
to need to be rewritten to work on substrings instead of on individual
codepoints. This, in turn, is going to require a heck of a lot more testing.

Last night I started putting in some of the changes necessary to use a
substring terminator instead of a single codepoint. Most of what I've already
done has been modifying function signatures. The real changes need to occur
deep within the buffering logic and will require a little bit more time.

I'm looking forward to getting this branch fixed up and merged back to master
so I can get to work on my next project. I think 6model is going to be the
next thing I dig into, before I find something else that annoys me enough to
put in a huge amount of effort to rewrite it. I'll post more updates about
my future projects and plans as I go.




