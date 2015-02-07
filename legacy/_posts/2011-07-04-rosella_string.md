---
layout: post
categories: [Parrot, Rosella, String]
title: Rosella String
---

I started writing a library of string-handling routines a while back, but only
worked on it casually and occasionally for most of it's life. I've finally
come across a need to have such a library around, so I've decided to start
putting some more effort into it.

Rosella String is a library of tools for working with strings. It exists for
several reasons:

1. Parrot does have many string-related opcodes and a few string-related PMC
   types. However, the semantics and interfaces to all these things are not
   always consistent nor intuitive.
2. Because there is so much inconsistency, There's a pretty good chance that
   things in Parrot could get deprecated, replaced, or improved. In that case,
   any project not using a tested abstraction layer will be broken. We don't
   have any plans to do anything like this in Parrot yet, but there are
   tickets floating around.
3. Parrot is missing some features of a robust string-handling package, such
   as configurable tokenizers. These are things that other projects (notably
   tcurtis' LALR project for GSoC) that could really benefit from having these
   things available.

For all these reasons, and because I felt like writing up the code, We now
have the Rosella String library.

At the moment, String is not a particularly full-featured library. It
currently contains two bits of functionality: A series of utility routines for
working with strings, and a library of text tokenizer classes. In the future
I intend to add other cool features like a Rope and a Trie type as well.
However with the current state of Parrot's object model, something like a
Trie type would be an exercise in inefficiency, bad performance, and futility.
I'm not going to write that code just to have it, but when it makes good sense
these kinds of things will be added.

So, what are we able to do with the string library? First, there are the
standard-looking utility functions:

    s = trim_start("  test");   # "test"
    s = trim_end("test  ");     # "test"
    s = trim("  test  ");       # "test"
    s = pad_start("test", 7);   # "   test"
    s = pad_end("test", 7);     # "test   "
    s = remove_start("test", 2);# "st"
    s = remove_end("test", 2);  # "te"
    i = indexof("test", "es");

...etc. There are several functions and the list grows everytime I have a
whim, a dream, or a really bad idea. These are pretty standard and, like I
said, some of them are very thin wrappers around opcodes in Parrot.

Next, we have tokenizers. Tokenizers break an input string up into individual
tokens according to various rules. Tokenizers, unlike something like the
`split` opcode, are lazy and return Token objects with type names and metadata.

Here's an example of the CClass Tokenizer, which breaks a string into chunks
with common character classes:

    my $t := Rosella::construct(Rosella::String::Tokenizer::CClass);
    $t.set_data("a = foo + bar;");
    while($t.has_tokens) {
        say("'" ~ $t.get_token.data ~ "'");
    }

The output of this will be the following:

    'a'
    ' '
    '='
    ' '
    'foo'
    ' '
    '+'
    ' '
    'bar'

Just with this simple tool, we already have most of what we need for
tokenizing a modern programming language. You would need a little bit of
extra logic to sort out a sequence of similar things which serve different
purposes. A sequence such as "i=((i++));" in C-like syntax would come out
of a naive CClass parser as such:

    'i'
    '=(('
    'i'
    '++));'

So that's not extremely helpful. You would need to look at all the punctuation
tokens and have to re-parse them to get meaningful operators and other
features.  With the CClass parser, you can specify a list of character class
codes to look for, and anything not included in that list will be returned
one character at a time. The input sequence 'foo++' without punctuation class
specified would come back as:

    'foo'
    '+'
    '+'

And once we have that output, we could use a secondary composer, such as a
Trie, to combine individual punctuation characters into multi-character
operators. Like I mentioned above, we don't have a Trie class available, but
a similar behavior could be done like NotFound did in the Winxed parser
source with nested hashes.

There are currently two other types of tokenizers in the String library:
A Delimiter tokenizer which splits up a string using delimiters (think about
a CSV file), and a DelimiterRegion tokenizer which uses start/end sequences
to extract regions from the input. Here's an example of a DelimiterRegion
tokenizer parsing through a simple string with Markdown-like markup and HTML
tags:

    my $t := Rosella::construct(Rosella::String::Tokeizer::DelimiterRegion, "Text");
    $t.add_region("*", "*", "Emphasis");
    $t.add_region("[", "]", "Link");
    $t.add_region("<", ">", "HTMLTag");
    $t.add_data("This *is* markup with [links] and <b>HTML</b>");
    while ($t.has_tokens) {
        my $k := $t.get_token;
        say($k.type_name ~ ": '" ~ $k.data ~ "'");
    }

...And the output:

    Text: 'This '
    Emphasis: 'is'
    Text: ' markup with '
    Link: 'links'
    Text: ' and '
    HTMLTag: 'b'
    Text: 'HTML'
    HTMLTag: '/b'

So that's not too shabby at all, for only 4 lines of code to create a
tokenizer and setting up a few named regions to look for.

So that's the Rosella String library. It's got limited usefulness now but I
do plan to add to it in the future. Tomorrow or shortly thereafter I'm going
to talk about the templating library I've been prototyping, which is a big
motivating factor behind this String library.
