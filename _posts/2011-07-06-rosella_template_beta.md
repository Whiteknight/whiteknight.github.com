---
layout: post
categories: [Parrot, Rosella, Template]
title: Rosella Template Beta
---

I mentioned it twice this week, once in a post about [Rosella Path][path] and
once in a post about [Rosella String][string]. Today I'm going to give the
first look at a new library I've been developing as part of the Rosella
project: Template.

[string]: http://whiteknight.github.com/2011/07/04/rosella_string.html
[path]: http://whiteknight.github.com/2011/07/02/rosella_path.html

Template is a text templating library drawing significant inspiration from
software like Liquid, ASP.NET, and other sources. It is one of a new class of
Rosella "high level" libraries, which build on several of the lower-level
foundational Rosella libraries. Originally the stated goal of the Rosella
project was to have a lot of small independent libraries, but with Template
that goal is starting to change. Now I plan to have libraries of multiple
levels, including low-level libraries which tend to be fundamental and
independent, and higher level libraries which build on the base functionality
to do bigger and better things.

Enough of the philosophy, let's look at some code. Using the templating
library is very simple. Currently, the code looks like this:

    var engine = new Rosella.Template.Engine();
    string output = engine.generate(template, context);

The variable `template` is the string template, and `context` is the data item
that provides information necessary to render the template. By default, I have
three different types of tags that can be used in a template: Data (`<# #>`),
Eval(`<% %>`) and Logic (`<$ $>`). This is just the default list with the
default tag delimiters, the user can add new types of tags, remove old tags,
or set custom delimiters for any of these. When the interfaces all stablize
and I'm ready to mark the library "stable", I'll talk more about how to do all
this customization. For now, I'm going to talk about the defaults only.

Data blocks take a single argument, a search path (using the Rosella Path
library) to get data out of the context. For instance, if this is my context:

    var context = { "Foo": { "Bar" : "hello!" } };

And if this is my template:

    I like to say "<# Foo.Bar #>"

The output will look like this:

    I like to say "hello!"

Eval blocks take nested code, compile it and execute it. By default the
compiler used is the winxed compiler, but the default compiler can be changed
or new tag types can be registered with new compilers (you can have multiple
HLL source types in a single template, with different tags for each). Inside
the eval, you have access to the variables `output` (a StringBuilder) and
`context` The context object. Here is an example of an Eval block in a
template:

    <%
        for (var x in [1, 2, 3, 4])
            push(output, sprintf("Item #%d\n", [x]));
    %>

And the output will look like this:

    Item #1
    Item #2
    Item #3
    Item #4

Eval templates can be dangerous if you're using them in an untrusted
environment, like the web. They can be disabled if you don't want them to be
available.

The last type of tag are logic tags. Logic tags come in a variety of different
types, and do all sorts of things. Here is a complex example:

    <$ for foo in bar.baz $>
        <# foo #>
        <$ if foo == "hello" $>
        Found a Greeting!
        <$ else $>
        Just some normal text
        <$ endif $>
        <$ include foo/bar.txt $>
    <$ endfor $>

This example is just a silly example, and the output depends on a lot of
factors. I'll give a few notes, however. First, the `for` loop looks up the
path `bar.baz` in the context object, iterates it, and stores subsequent
values in a temporary variable named `foo`. The contents of the loop are
repeated, each time with a new value. The `if`/`else` block can compare
variables with other variables, variables with constants, constants with
constants, etc. It has a then and optional `else` block. There is also an
`unless` block too, which I didn't show, but operates opposite to `if` as you
would expect. The `include` block includes the text of a separate file,
recursively parsing it as a template using the same context object.

Loops have a few magic variables that are defined inside them:

    <$ for foo in bar.baz $>
    <$ if __FIRST__ $> ( <$ endif $>
        <# foo #>
    <$ unless __LAST__ $> --- <$ else $> ) <$ endunless $>
    <$ endfor $>

If I have this context object:

    var context = { "bar" : { "baz" : [1, 2, 3] } }

... I'm going to get output like this (the whitespace in the template is *not*
ignored):

    (
        1
        ---
        2
        ---
        3
    )

The magic variables `__FIRST__` and `__LAST__` resolve to true or false if we
are at the first or last item in the loop, respectively. Also, if we are
iterating a hash, the magic variable `__KEY__` will contain the name of the
current hash key (`foo` will contain the value). If I use the for loop with a
scalar value (not an array or a hash), it will iterate just once, and both
`__FIRST__` and `__LAST__` will be true.

Those are the basics of the Template library. I'm already starting to put
together a small collection of templates and even a utility program for
generating them. One program I'm putting together right now is used to create
stub test files from a test template. From the Rosella directory, I can do
this:

    ./test_template test --lang=winxed --lib=rosella/string.pbc --class=Rosella.String.Tokenizer

That program will load in the given library, get the given class object,
get a list of all methods and vtable overrides in that class, and will create
a stub test file in the given language with Rosella Test goodness. Currently,
I have templates for NQP and Winxed written. I may put together a template for
PIR or other languages too.

Another thing I can do is this:

    ./test_template harness --lang=nqp

That will automatically create a new test harness in NQP. These two bits of
functionality together make testing a snap. This is just the start, of course.
As far as testing is concerned, I want to start putting together templates for
tests with MockObject as well, although I'm not sure what I want the interface
to look like.

I want to put together many other types of standard templates, like distutils
setup program templates, templates for doing parrot-related tasks (PMC source,
op library source, Test::More-based PIR tests, Parrot API function templates,
etc. Basically, if you're doing work on Parrot, I want to have text templates
around to help make your work go faster.

I've got a lot of work left to do on this library. The internal logic is not
nearly in the form I want it to be in, and it's not nearly as customizable
yet as I would like it to be. I am starting to put together tests for the
functionality I like, and am planning for the changes I want to make. I don't
want to give any kind of timeline for this work, but I am pretty excited about
this new library and will probably focus on it a lot more in the next few
days, especially if we have a long code freeze before the 3.6 release. I need
to wrap up my current Packfile branch and start planning for my next big
parrot-related project (probably 6model), but between now and then I am going
to be playing with this.

