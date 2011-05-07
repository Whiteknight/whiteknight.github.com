I had been planning to get to work on some of the things that Rakudo had
requested involving Packfiles next, but after talking to plobsing we've
decided to hold off on those things for now and push them off until towards
the end of summer. He instead is interested in starting to do some concurrency
work now, and I'm going to set my sights on the meta-object model instead,
along with offering a little bit of help for the concurrency project.

The general priorities for the 4.0 release, roughly sorted in order of both
what we want and what I think people are actually going to be working towards,
are as follows: basic concurrency work and thread-safety (plobsing and
myself), 6model and MOP improvements (I'm trying to assemble a team. Currently
it's Tene and myself), packfile improvements (plobsing and myself), Lorito
prototypes (cotto and dukeleto), and completing the work to excise IMCC from
libparrot (I *need* this for my own personal sanity). In addition, bacek is
still working hard on his JIT prototype work, and we have a series of very
interesting GSoC projects which need our support. When you start adding up the
things that we can plausibly deliver before 4.0, it really amounts to quite an
impressive amount of stuff. I hope we do half as well.

Today I'm going to talk about the meta-object model work that I am planning to
start soon. This is the first of what will indoubtably become a lengthy series
of posts on the topic, and is likely to be the least complete and most
ignorant from among them.

The basic plan for our object model going forward is to adopt 6model as the
basis of a new flexible system. It and the things we will build on top of it
are mostly complete in terms of the necessary high-level functionality that
our HLLs need. Because it was designed to run on top of Parrot and to cater to
the needs of Perl 6 it doesn't quite have everything that we need out of the
box. However, it does give us a great opportunity to throw out the baby with
the bathwater and clean up some of the biggest problems in our current system.
We also need to be cognizant of the fact that the object model refactors are
going to take a long time and be very involved, and that they will be occuring
before, during, and after several other big changes to other important
subsystems. The system is not being refactored and rewritten in a vacuum, we
need to be aware of work in the following subsystems: GC, JIT, Concurrency,
Lorito and Packfiles, among others.

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

Interestingly enough, chromatic posted a very related post about [this very
topic on his blog][chromatic_post].

[chromatic_post]: http://www.modernperlbooks.com/mt/2011/05/when-you-lack-cheap-and-easy-polymorphism.html

I'm not convinced that chromatic is talking to a Parrot-centric audience in
that post. In fact, I'm very convinced that he is holding Parrot up as an
example of what not to do to a Perl5-centric audience. No matter. We can all
learn from the bad example Parrot is setting. Let's face facts: The current
Parrot object model can be summed up with two simple adjectives: stagnant and
disappointing.

Fixing the object model and improving it so it can be better in the future are
both extremely important tasks for the viability of Parrot. We need object
model improvements in a very bad way. 6model is a kind of low-level tool that
we would like to insert as the basis for a new system and the kinds of radical
improvements that I think we want and need. In subsequent posts, as I get more
concrete ideas in my head about what we need to do and what order we should
try to do them in, I'm going to be filling in some of the details. The hard
part in all of this, as I mentioned above, is trying to keep track of the work
and the planning that is going into GC, concurrency, and Lorito. We are all
going to have to try extremely hard to make sure all these fancy new systems
work well together, and aren't designed separately without accomodation.



