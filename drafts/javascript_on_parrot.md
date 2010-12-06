---
layout: post
category: [Parrot, JavaScript]
title: JavaScript in JavaScript, on Parrot
---

There exists a compiler for JavaScript on Parrot. The project, the horribly
named "ECMAScript" project, is quite old and has fallen into disrepair. It
relies on the old NQP and PGE, and has only recently been fixed to build
again. It still fails several tests and there aren't too many people in line
to attempt fixes.

JavaScript is a pretty popular language, and is slowly becoming something of
a *lingua franca* of portable scripting languages. JavaScript is supported in
almost every single modern web browser and many applications are starting to
embed interpreters to add JavaScript scripting capabilities. It still doesn't
have great support at the commandline, but that situation is slowly changing.

Rhino is the current command-line solution on Linux. I've never really heard
much about it but what I have heard typically involves complaints.
Performance, especially startup performance gets some negative reviews, and
the fact that it's written in Java and only really has bindings to things
running on the JVM also gets some complaints. Node.js uses Google's speedy
V8 engine under the hood, but I haven't heard many reviews about it (good or
bad). Node.js does appear to have a certain focus on event-drivin IO
applications, though I don't know to what degree that focus helps or impairs
applications which run atop it.

There is definitely a market for a command-line JavaScript interpreter,
especially one which is speedy, feature-full, and interoperable with other software. I think a JavaScript interpreter on Parrot can be a nice offering in
this area, but the current incarnation might not be the thing we want to put
forward.

Let's take a step back and take a look at Parrot's support for compilers. I've
said on previous occasions that PCT really is a kind of "killer app" for
Parrot: It allows quick prototyping and development of powerful
next-generation compilers. I've seen individuals put together prototype
language compilers using PCT in less than a week for real languages. It is
an impressive productivity-boosting tool if you know hot to use it, and PCT
provides many cool features that compiler hackers would really love.

The downside for many people is that PCT is written in Perl6. This isn't a
complaint against Perl 6 (it's a language that I very much enjoy). We need to
take yet another step back and take a look at the context of things.

Developing the Rakudo Perl6 compiler in a subset of Perl6 is quite a cool
thing, and is in part a reason for the popularity and development progress
of that project. Perl6 is large and ambitious compared to many other languages
and without an enabling tool like NQP behind it I do not believe that the 
Rakudo team would be where they are today. Developing Perl6 in Perl6 is, to
a Perl6 hacker, a great thing.

Developing Python in Perl6 to a Python hacker, on the other hand, is not so
cool. A dedicated Python hacker probably knows Python and maybe some C/C++.
They probably don't know Perl, or if they do know it they probably don't like
it.

Back to the topic, let's translate this idea to JavaScript. People who want
a JavaScript-on-Parrot compiler, and who have the knowledge of the language
to create the compiler in the first place, probably know JavaScript. In fact,
that's probably a safe assumption. What we cannot assume is that a
JavaScript-on-Parrot compiler hacker knows (or even likes) Perl6. What people
would really like to do, I think, is to write the JavaScript-on-Parrot
compiler in *JavaScript*. I also think that Parrot is mature enough now that
we can support that.

How?

Bootstrapping is a funny thing, but is also a well-understood problem. GCC
is written in C, and they've put out dozens of releases without accidentally
creating a paradox and destroying the universe. The [PyPy][] project wrote
a Python compiler in Python with some pretty hot new technology thrown in as
well. There exist tools written in JavaScript to parse JavaScript. Two
projects that have my attention in particular are [jison][], an LALR parser
generator written in JavaScript, and [cafe][] a JavaScript parser written
using jison. Once we have either these tools, or something like them at our
disposal, the steps to creating a bootstrapped JavaScript compiler in
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

At this point you have a fully bootstrapped JavaScript-in-JavaScript-on-Parrot
compiler. All future changes to the compiler can be made quite easily:

1. Make a change to the JavaScript source of the compiler
2. Compile it with your current Stage 1 compiler
3. Run all tests with the new version to verify performance
4. Replace your current Stage 1 compiler with the new compiler.

Easy to do and easy to automate. Once we have the basic compiler in place we
can start implementing JavaScript optimizers in JavaScript, analysis and
debugging tools in JavaScript, unit tests in JavaScript, etc. In short, a
good JavaScript developer becomes 100% productive because all the code he
writes from the bottom to the tippy-top is written in his language of choice.

PCT is a great tool, and if you know and enjoy Perl6 I definitely recommend
you give it a try for prototyping and implementing your new language compiler.
In some cases, it's Perl6 roots really just aren't right for a project and
actually become something of a barrier to entry for new developers. In those
cases, where alternative parsers exist, we can start talking about a
bootstrapped solution instead. Take an existing parser, create a PIR code
backend for it to get started, and you're well on your way to writing your
favorite language for a compiler written in your favorite language, which you
also wrote.

It doesn't take any level of genius or a PhD in computer science to start 
using any of these fun ideas, all it takes is Parrot and a few hours of your 
time.

