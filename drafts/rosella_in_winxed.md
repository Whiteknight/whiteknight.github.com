Rosella is still a very young project, but a few days ago I decided to
basically throw out all the existing code and rewrite it in winxed. Winxed,
for those readers who aren't familiar with the entire Parrot ecosystem, is a
JavaScript-ish language written by core developer NotFound. Winxed borrows
some syntax and ideas from JavaScript (and, occasionally, C++), and aims to
be a very low-level and light-weight language for use with Parrot.

The first version of Rosella was written in NQP, the perl6 subset language
which was developed both as a tool for writing Rakudo but also as a general
tool for writing HLL compilers on Parrot (and, eventually, other VMs as well).
NQP is very good for this purpose, and I would recommend it for almost any
HLL compiler project on Parrot, sight unseen.

This begs the question of course: Why am I rewriting Rosella in Winxed? Even
though Rosella is young the code base is not trivial, and rewriting it is
proving to take some serious concerted effort.

Here's the part of the post where I should say the things I don't like about
NQP. Follow that up with a glowing review of Winxed and then a nice conclusion
to reemphasize all the previous points, and the choice of language should be
clear to all readers. I'm not going to follow that path, however, because
I don't have any major complaints about NQP, and it serves its purposes very
nicely. Also, Winxed isn't perfect itself, but is in active development (and
NotFound has been extremely responsive to bug reports and feature requests).

What the decision came down to was a matter of thinking that one language was
aligned a little bit more closely with my goals for Rosella than the other
language was. I think that Winxed makes just a little bit more sense for
Rosella in the long run, and I want to get this decision as right as possible
now while it's still early in life.

NQP has certain semantics required of it through its Perl6 roots. These
behaviors manifest themselves in the classes P6object, P6metaclass, and
P6protoobject. To the average user of NQP these classes are mostly
transparent. You write NQP code and it does exactly what you would expect it
to do. The underlying mechanisms that provide these behaviors are based on
these three classes.

P6object and friends provide a lot of great behavior and functionality which
is available even if you don't particularly need it. I don't want to use the
term "bloat" here, because the implementations of P6object and P6metaclass
aren't particularly large and don't add much overhead to program execution.
It still is an extra dependency that we need to be aware of. If we run the
library through the `parrot-nqp` binary, the P6object library (`P6object.pbc`)
is included automatically. If we use a different front-end, it isn't.

Here's some example NQP code for loading and using the Rosella TAP Harness
library (like I showed in a previous post):

    INIT { pir::load_bytecode("rosella/tap_harness.pbc"); }
    my $h := Rosella::build(Rosella::Harness);

This toy example uses Rosella's built-in initializer routines to create a new
harness object. No magic here, nothing fancy. Now, let's write this same
exact functionality in Winxed:

    function main[main]() {
        load_bytecode("rosella/tap_harness.pbc");
        using Rosella.build;
        var s = build(class Rosella.Harness);
    }

We run this and...Blamo. It fails. Internally the Rosella library is using
some features of P6object, but we haven't loaded `P6object.pbc` yet. After I
mentioned this issue on IRC, I think some people either suggested or
implemented a fix for this problem in the NQP source. By the time this post is
published this might not really be a big blocking issue anymore.

What I want for Rosella is to be a portable set of low-level utilities,
libraries, and building blocks. I do not want Rosella to be relying on any
other libraries as dependencies, not even libraries that ship with Parrot
directly. I want Rosella to be language agnostic. I want Rosella to be able
to work with Parrot built-in types just as easily as user-defined classes,
and P6object-based NQP classes. Most of these goals can be achieved with NQP,
although many of them can be achieved a little bit better with Winxed. It's a
little bit splitting of hairs, a little bit personal preference, and a lot of
fun writing cool new software in cool new languages.

To see the language agnosticism at work, you don't need to look any further
than Rosella's own test suite. Most of Rosella's libraries have been rewritten
in Winxed, although the unit testing library is still written in NQP. Also,
the test harness driver program is written in NQP. So we have an NQP driver
program including the `rosella/tap_harness.pbc` library (written in Winxed),
running test cases written in NQP, which include the `rosella/xunit.pbc`
library (written in NQP, but which itself includes other rosella libraries
written in Winxed) to test Rosella libraries written in Winxed. And what's
most spectacular about this whole situation is that there are no
cross-language issues to worry about. Everything works seamlessly. Parrot,
for the win!

Parrot-Linear-Algebra (PLA), my linear algebra package for Parrot, uses
Rosella for its test suite and harness, but the harness driver program and all
the individual test files are written in NQP. That isn't going to change.

When I finally (if ever) get back to working on Matrixy, my MATLAB/Octave
clone language compiler, I'll still be writing most of that in NQP (though I
will almost certainly be using several bits from Rosella to assist).

Rosella's XUnit library is still written in NQP. So are a few of the newer
stub libraries that I've been exploring with. By this weekend I expect Rosella
to be completely rewritten in Winxed. I'll be keeping the old NQP versions of
the libraries around, for a while at least. I probably won't be backporting
any new features. If you want the newer, the bigger, and the better, you're
going to want to use Winxed instead.
