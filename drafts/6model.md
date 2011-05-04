I had been planning to get to work on some of the things that Rakudo had
requested involving Packfiles next, but after talking to plobsing we've
decided to hold off on those things for now and push them off until towards
the end of summer. He instead is interested in starting to do some concurrency
work now, and I'm going to set my sights on the meta-object model now instead,
along with offering a little bit of help for the concurrency project.

The general priorities for the 4.0 release, in order of both what we want and
what I think people are actually going to be working towards, are as follows:
basic concurrency work (plobsing and myself), 6model and MOP improvements
(I'm trying to assemble a team. Currently it's Tene and myself), packfile
improvements (plobsing and myself), Lorito prototypes (cotto and dukeleto),
and completing the work to excise IMCC from libparrot (I *need* this for my
own personal sanity). In addition, bacek is still working hard on his JIT
prototype work, and we have a series of very interesting GSoC projects which
need our support. When you start adding up the things that we can plausibly
deliver before 4.0, it really amounts to quite an impressive amount of stuff.

Today I'm going to talk about the meta-object model work that I am planning to
start soon. This is the first of what will indoubtably become a lengthy series
of posts on the topic, and is likely to be the least complete and most
ignorant from among them.

The basic plan for our object model going forward is to adopt 6model. It and
the things we will build on top of it are mostly complete in terms of the
necessary high-level functionality that our HLLs need. Because it was designed
to run on top of Parrot and to cater to the needs of Perl 6 it doesn't quite
have everything that we need out of the box. However, it does give us a great
opportunity to throw out the baby with the bathwater and clean up some of the
biggest problems in our current system. We also need to be cognizant of the
fact that the object model refactors are going to take a long time and be very
involved, and that they will be occuring before, during, and after several
other big changes to other important subsystems. The system is not being
refactored and rewritten in a vacuum, we need to be aware of work in the
following subsystems: GC, JIT, Concurrency, Lorito and Packfiles, among
others.

The basic high-level plan to bring 6model into Parrot goes like this: We bring
6model into Parrot, in parallel with our current object system. Then we start
migrating our current system to run on top of 6 model. Then we start tearing
down the walls and exposing 6model more directly to the user. Along the way we
want to fix a huge number of problems and inefficiencies throughout Parrot.
The way I figure it, if we're going to do this project we should do it right.

Some of the problems I really would like to address are: inheritance between
built-in types and PIR-defined types, multi-pass PMC type initialization at
Parrot startup, vtable bloat, various inefficiencies with attribute storage
(especially in Object), improvement of Class and Role types, including a real
ability to use roles for built-in types, and a handful of other things.



