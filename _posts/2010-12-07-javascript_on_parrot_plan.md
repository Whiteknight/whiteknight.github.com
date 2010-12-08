---
layout: post
category: [Parrot, JavaScript]
title: JavaScript in JavaScript, on Parrot
---

There exists a compiler for JavaScript on Parrot. The project, the horribly
named "[ecmascript][]", is quite old and has fallen into disrepair. It
relies on the old NQP and PGE, and has only recently been fixed to build
again. It still fails several tests and there aren't too many people in line
to attempt fixes.

[ecmascript]: http://github.com/parrot/ecmascript

JavaScript is a pretty popular language, and is slowly becoming something of
a *lingua franca* of portable scripting languages. JavaScript is supported in
almost every single modern web browser and many applications are starting to
embed interpreters to add JavaScript scripting capabilities. Not nearly so
many non-browser applications are embedding it as are embedding Pythong, but
it's still gaining a niche slowly but surely in this role. JavaScript still
doesn't enjoy great support at the commandline, but that situation is slowly
changing too.

Rhino is the current command-line solution on Linux. I've never used it and
haven't really heard much about it but what I have heard typically involves
complaints. Performance, especially startup performance gets some negative
reviews, and the fact that it's written in Java and only really has bindings
to things running on the JVM also gets some negative marks. Node.js uses
Google's speedy V8 engine under the hood, but I haven't heard many reviews
about it (good or bad). Node.js does appear to have a certain focus on
event-driven IO applications, though I don't know to what degree that focus
helps or impairs applications which run atop it.

There is definitely a market for a command-line JavaScript interpreter,
especially one which is speedy, feature-full, and interoperable with other
software. I think a JavaScript interpreter on Parrot can be a nice offering in
this area, but the current incarnation might not be the thing we want to put
forward. In fact, I definitely don't think that the ecmascript project on
Parrot is going to do much to improve our popularity at all. Just call it a
hunch that a poorly-branded implementation with no active maintainers, which
is written in an old version of a subset of Perl6 isn't going to garner much
support. Just a hunch.

Let's take a step back and look at Parrot's support for compilers. I've
said on previous occasions that PCT really is a kind of "killer app" for
Parrot: It allows quick prototyping and development of powerful
next-generation compilers. I've seen individuals put together prototype
language compilers using PCT in less than a week for real languages, much
faster for toy languages. It is an impressive productivity-boosting tool if
you know how to use it, and PCT provides many cool features that compiler
hackers would really love.

By all respects, I think it's one of the easiest-to-use and feature-full
parser utilities available. We can be honest about the performance of
the parser and of the generated code not being very good, but that's a work
in progress.

The downside for many people is that PCT is written in Perl6. This isn't a
complaint against Perl 6 (it's a language that I very much enjoy personally).
But we do need to take yet another step back and take a look at the context of
things. If everybody loved Perl absolutely, there might not be languages like
PHP, Python, Ruby, or JavaScript in existence. Perl certainly beat all those
comers to market.

Some people love Java, others hate it. Some people love C++, others hate it.
Some people love Haskell, others hate it. Some people love Lisp, others hate
it. This is the nature of software, or any other area of human enterprise
where people are able to make decisions based on subjective reasoning. Some
people really dig Perl, some people absolutely do not want to use it.

But beyond liking or disliking Perl 6 or PCT, there's the other idea that
people may just like a different language *more*. Or, people may know a
different language *better*. Telling people who know JavaScript that they
need to write their compiler in Perl 6 might be something of a hard sell. I
know for a fact that Pynie, the Python-on-Parrot compiler, suffers for this
kind of reason: Python people want to write Python, not some other language.
They certainly do not want to be writing Python in Perl. Of course, for the
Python people, this might be cultural.

Developing the Rakudo Perl6 compiler in a subset of Perl6 is quite a cool
thing, and is in part a reason for the popularity and development progress
of that project. Perl6 is large and ambitious compared to many other languages
and without an enabling tool like NQP and PCT behind it I do not believe that
the  Rakudo team would be where they are today. Developing Perl6 in Perl6 is,
to a Perl6 hacker, a great thing.

For people who know Perl and maybe even like it, want to create a compiler for
a new language, and want to do it quickly, PCT and NQP are great tools and
absolutely fit for the job. For cases where an established language and
decent compilers/interpreters for it already exist, where the developers don't
know Perl or don't like it, and for which suitable compiler toolkits exist
in that language, maybe bootstrapping is a better option. I certainly think it
might be a great option for JavaScript.

Back to the topic, let's talk more about this bootstrapping JavaScript idea.
People who want a JavaScript-on-Parrot compiler, and who have the knowledge of
the language to create the compiler in the first place, probably know
JavaScript. There exist JavaScript interpreters already that we could use to
begin a bootstrapping project. There even exist parser-generator utilities,
written in JavaScript, which can be used to parse JavaScript.

See where I am going with this?

Bootstrapping is a funny thing, but is also a well-understood problem. GCC
is written in C, and they've put out dozens of releases without accidentally
creating a paradox and destroying the universe. The [PyPy][] project wrote
a Python compiler in Python with some pretty hot new technology thrown in as
well. If they can do it, JavaScript-on-Parrot can do it too.

[PyPy]: http://pypy.org

Two projects that have my attention in particular are [jison][], an LALR
parser generator written in JavaScript, and [cafe][] a JavaScript parser
written using jison. Once we have either these tools, or something like them
at our disposal, the steps to creating a bootstrapped JavaScript compiler in
JavaScript are extremely simple:

1. Write a PIR code generating backend to Cafe, or any other existing
   JavaScript-in-JavaScript parser (or, write our own parser too, but that
   is probably way too much work considering the availability of existing
   solutions).
2. Run this program on an existing JavaScript engine like Rhino to produce the
   PIR code of a working JavaScript interpreter
3. Use Parrot to compile down the PIR code to bytecode and link it into a
   stand-alone executable. This is called "Stage 0".
4. Use Stage 0 to run all tests and verify behavior
5. Use your new executable to compile it's own source code. This second
   version, compiled using itself, is called "Stage 1"
6. Use Stage 1 to run all tests and verify behavior

[cafe]: http://github.com/zaach/cafe
[jison]: http://jison.org

At this point you have a fully bootstrapped JavaScript-in-JavaScript-on-Parrot
compiler. All future changes to the compiler can be made quite easily:

1. Make a change to the JavaScript source of the compiler
2. Compile it with your current Stage 1 compiler
3. Run all tests with the new version to verify performance
4. Replace your current Stage 1 compiler with the new compiler.

Easy to do and easy to automate. Once we have the basic compiler in place we
can start implementing JavaScript optimizers in JavaScript, analysis and
debugging tools in JavaScript, unit tests in JavaScript, etc. In short, a
good JavaScript developer becomes 100% productive because all the code she
writes from the bottom to the tippy-top is written in her language of choice.

PCT is a great tool, and if you know and enjoy Perl6 I definitely recommend
you give it a try for prototyping and implementing your new language compiler.
In some cases, it's Perl6 roots really just aren't right for a project and
actually become something of a barrier to entry for new developers. In those
cases, where alternative parsers exist, we can start talking about a
bootstrapped solution instead. Take an existing parser, create a PIR code
backend for it to get started, and you're well on your way to writing your
favorite language for a compiler written in your favorite language, which you
also wrote.

This is exactly what I think we can do for JavaScript, by leveraging existing
tools and libraries to get started. With a few developers and a little bit of
effort I think we could have a mostly-complete JavaScript parser, written in
JavaScript, running on Parrot in only a few months. It's an interesting goal
and one that I think could help improve the prestige of Parrot as a platform
for these kinds of things.

