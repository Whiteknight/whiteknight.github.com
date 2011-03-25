


## Problem Statement

This is two separate projects, which I am grouping together here into a single
post:

> Create an LALR parser generator tool. The tool should be written in one of
> the lower-level languages which run on Parrot (PIR, NQP, Winxed, other) for
> maximum portability. The parser generator should take a language
> specification and compile it into an LALR parser in a manner similar to
> YACC/Bison. The generator parser likewise can be in any suitable language.
> For this project, a working LALR parser generator engine is more important
> than the ability to understand a huge number of input patterns and pattern
> modifiers.

> Create a new code generation backend for ANTLR to produce parsers which run
> on Parrot. Target languages can be any language which runs on Parrot,
> although lower-level languages are preferred for performance and portability
> (PIR, NQP, Winxed, etc). The backend should be able to create working code
> for the full range of ANTLR features.

In short: Parrot has the recursive decent parser which is part of NQP, and it
has a port of OMeta for Winxed. We would really like to have access to other
parser technologies, especially popular ones which would be more familiar to
developers of existing compilers.

### Other Parser Types

The two I mentioned above are just two ideas which could be used. There are
other types of parsers which could be implemented for Parrot as well:

* SLR Parsers
* GLR Parsers
* CYK Parsers
* Earley Parsers

Any of these might be interesting things to attempt to implement on Parrot,
although they would probably have less usefulness than ANTLR or LALR that
I mentioned above.

### Other Related Projects

Instead of working on a parser, a lexer/tokenizer (Similar to Lex/Flex)
might be an interesting project, and would be able to be used by many other
projects for a variety of purposes. For instance, a tokenizer library could
be useful not only in the front-end of compilers, but also for other string
manipulating routines.

If you're a real theoretical type, are in gradschool for this topic or are
planning to attend, something related to parser derivatives and parser
combiners might make for an interesting project. I suspect we start getting
into the realm of difficulty which is too large for a single GSoC summer, but
if the right person picked the right project there could be major benfits
for both Parrot and for the academic future of the student.

Implementing a powerful regular expression engine on Parrot might be nice too.
NQP uses the Perl6 flavor of regular expressions, but many other languages
use the older "Perl 5" style instead, which Parrot has very limited support
for. A regular expression engine based on the Perl5 syntax which was written
in pure Parrot and could be configurable enough to be used by other compilers
on Parrot (JavaScript?) would make a great project with huge benefits.

## Difficulty

ANTLR *Difficulty*: 2/5

I've done precious little research into ANTLR, but it seems to me from reading
existing examples that creating a backend for it is not entirely difficult.
There's a huge matter of having to completely test the solution, but following
existing solutions and substituting in target code for a Parrot language
instead should probably not be too difficult.

Here is an interesting page to read to get started with building a new backend
for ANTLR: [How to build an ANTLR code generation target][antlr_how_to].

[antlr_how_to]: http://www.antlr.org/wiki/display/ANTLR3/How+to+build+an+ANTLR+code+generation+target

LALR *Difficulty*: 5/5

LALR I think is going to be a more difficult project to implement, because
you would be building the project from the ground-up, not just implementing
a backend for an existing system. In many ways this could be a simple case of
finding an existing LALR parser generator and doing a line-by-line translation
of the algorithm. Of course, the term "simple" in the previous sentence is
extremely relative.

Where we gain a little bit of wiggle room from this project is that you could
more easily implement a subset of the LALR algorithm, or only a subset of
patterns to use as input, with the hope that the system could be extended
later. With ANTLR, it's harder to demarcate a subset, you would probably just
need to fill in all the blanks.

As a possible way to ease the requirements of this project would be to create
an engine for the GOLD parser generator, which is an LALR system.

## Deliverables

The real motivation behind the LALR and ANTLR parser projects is to be able
to easily port existing language parsers to Parrot.

If you pick the ANTLR project, I think I would personally like to see the
ability to create a working parser from a non-trivial existing grammar by the
end of the summer. This isn't a hard requirement, just something I would like
to see if possible.

If you pick the LALR project, I would like to be able to use your tool to
generate a correct LALR parser for trivial cases (an RPN calculator?), and
also see some documentation/examples about how to extend the project to add
in new features when the summer is over.

In both cases all the standard deliverables apply: The necessary build
infrastructure, a comprehensive unit test suite, ample documentation and
plenty of code examples to help get other coders involved in the project.

## How to Get Started

If you want to do ANTLR or LALR, grab a copy of the necessary code and start
using/testing/understanding it. For ANTLR, get that code. For LALR get a copy
of Bison, GOLD, JavaCC, Lemon, etc. If you can get your hands on a copy of it,
now might be a great time to start reading through the Dragon Book.

Get in touch with some Parrot developers also to start talking about ideas.
There are a number of possible mentors for these projects, so come to the
mailing list or the IRC chatroom and start talking so we can find the right
people to match you up with.

## Who Should Apply

Parsing is one of the areas of computer programming is the most abstract and
based in theory. If you don't have a background in parsing, and aren't a
quick study, you could get in over your head pretty quickly with any of these
projects.

If you aren't already familiar with concepts and many of the related topics I
recommend you avoid this project.

If you do have the necessary background, are a hard worker, and aren't afraid
of taking on challenging projects this could be the right one for you.
