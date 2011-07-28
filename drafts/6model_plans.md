I've been talking occasionally to Jonathan Worthington, Rakudo hacker and
author of 6model, what his roadmap looked like. Several weeks ago he told me
that the Rakudo refactor would likely require several changes to 6model
internals and it's API, and I should not try to migrate it in until things
had settled down. A few days ago I asked him again, and he replied that he
had one last big change to make: The removal of two PMC types from the
implementation which turned out to be superfluous. Earlier this week, he
removed one of them.

It's time to start thinking about 6model. It's time to start planning to put
it into Parrot. I could start the preliminary branchwork as early as next
week, depending on several factors, so now is as good a time as any to plan.

6model is, as has been impressed upon me many times, not an object model in
itself. It is, instead, a toolkit for creating object models. It is the
underlying framework on which we can build fast, flexible and efficient object
models to suite the needs of our users. It is the framework on which our users
can build their own object models, if what we provide isn't right. Chances
are, it isn't.

The first step in the 6model migration will be moving the source code files
into the Parrot repo, adding them to the makefile, and getting the whole mess
to build. This is both easy and not: 6model wasn't written under the Parrot
umbrella and is known not to build and run on all the platforms that Parrot
says are supported. Just getting 6model to build with a variety of compilers
will be a small task in itself.

Once we have the 6model code building, we need to make it do something. We
need to start exposing it to the user, and testing the hell out of it.
Initially, I think 6model will be separate from our current object metamodel,
available separately and in parallel from our current offerings, and the
Object and Class PMCs. Basically, the first phase of 6model-on-parrot will
be very similar, architecturally, to how it currently lives in NQP. Once we
have it in place, we can start making changes to get code to use it instead of
Object and Class. We can start migrating NQP to use the Parrot version instead
of its own version (eventually; we know that Rakudo is still not ready for
that level of committment and deprecation-policy-demanded stability), we can
migrate Winxed to use it instead, and we can start changing other standard
libraries to use it.

Eventually, we can start integrating 6model more deeply into Parrot, and we
can start switching things like Object and Class to be built on top of 6model
instead of competing with it. Object and Class will become just one possible
object model written on it and, as I think we will come to see, an inferior
one from among the other available options.

Once we have 6model in place, we can start building a new object model on
top of it. We can build one that takes better advantage of the 6model
efficiencies and capabilities. When people are ready, we can start migrating
over to use the new models, and deprecate Object and Class (and PMCProxy too!
good riddence!).

