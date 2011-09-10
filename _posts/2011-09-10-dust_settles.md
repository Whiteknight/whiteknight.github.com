---
layout: post
categories: [Parrot, Community]
title: Parrot, the Smoke Clears
---

For anybody who missed it, Parrot Architect Christoph sent [an email][email]
to the parrot-dev mailing list suggesting that things were not going well for
the Parrot project and we need to make a few changes.  Specifically, we need
to become faster and more nimble by ditching the deprecation policy, being
less formal about the roadmap, and being more focused on making Parrot better.
Later, Jonathan Leto sent out an email talking about how we need to have a
concise and clear vision to pursue, to prevent us from getting stuck in the
same kind of swamp that we're trying to pull ourselves out of. He had some
good ideas there that need to be addressed.

[email]: http://lists.parrot.org/pipermail/parrot-dev/2011-September/006179.html

That's right, go back and read it again: We're ditching the deprecation
policy. I'll bring the champagne, you bring the pitch forks and lighter fluid.
It's time for celebration. At least, it should be.

After the email from cotto went out, things went in a direction I didn't
expect. People started getting angry, and some of that anger was directed
towards Rakudo. I think it was misdirected. **Rakudo isn't the problem and
never has been**. The problem was the deprecation policy and some of the
related decisions that have been made with it over time.

The thinking goes, I think, something like this: The deprecation policy was
bad. Rakudo expected us to do what it said and that we promised to do.
Therefore, Rakudo must have been bad also. I'm oversimplifying, of course.

Rakudo has their own thing going on. They have goals, and they make long-term
plans and they have developers and they have dependencies and all the other
stuff that an active software project has. Parrot is a pretty damn big part of
their world, and knowing what Parrot is doing or what it plans to do and at
what times is important for them. If Parrot has a policy that says "we
guarantee certain things cannot change faster than a certain speed, or more
often than certain limited times", they start to make plans around those
guarantees and start to organize themselves in a way to take advantage of
that. It's what any project would do.

Imagine, as a Parrot developer, that the GCC developers sent out a mass email
tomorrow that said the equivalent of "Oh, we're not supporting the C89
standard anymore, we're only going to be compiling a custom language similar
to C but with some new non-standard additions, but without some of the
trickier, uglier parts of the standard. Nothing you can do about it, so get
to work updating your code if you want to continue using GCC". We'd have some
work to do, I'm sure, and we probably wouldn't be too happy about it. Rakudo
on the other hand knows that Parrot isn't nearly so mature or stable as GCC,
and has a lot of improvements to make. They might not always get credit for
it, but they have been pretty patient with Parrot over the last few years,
even when we probably didn't deserve so much patience.

Some people have said that Rakudo has been an impediment to Parrot
development, or that they are a reason why Parrot has problems and why the
pace of development has been so slow. I think that's a short-sighted
sentiment. It's not Rakudo's fault that they expect us to mean what our
official policy documents say. It's also not their fault that we put together
the deprecation policy in the first place, or that we've implemented it in
the particular way we have over the years.  In short, whatever negative
feelings some parrot devs think they have about Rakudo is just a smoke screen.
The way Parrot and Rakudo interact is the *symptom*. There are larger
cultural aspects at the root of the problem, and the deprecation policy was a
large part of that.

Rakudo developers, by and large, want Parrot to develop and improve *faster*.
I haven't spoken to a single Rakudo developer who was unhappy to see the
deprecation policy go. Most of them are ecstatic about the change. It's hard
to say that these people somehow want to sabotage us, or delay us, impede us,
or whatever else. The things that Parrot needs (better performance, better
encapsulation and interfaces, better implementations, better focus) are all
things that are going to benefit Rakudo as well. This is what they want too.
We're always going to quibble over details, but in general they want from us
what we want from ourselves. 95% of the improvements we need to make for
Rakudo are going to benefit other languages as well. The other 5% can be
negotiated.

Parrot and Perl6 have a pretty long and interesting history together.
Unfortunately, that history hasn't always been pretty. Parrot was originally
started as the VM to run Perl6. The idea of running multiple dynamic languages
in interoperation harmony, including early plans to support later versions of
Parrot 5, came later but eventually eclipsed the original goal in importance.
You don't have to look far, even in some of the subsystems that have been most
heavily refactored in recent months, to see Perl-ish influences in the code.
Sometimes those influences are far from subtle. You also don't have to look
too far to find instances of subsystems that were designed and implemented
(and redesigned, and reimplemented) specifically to *avoid* doing what Perl6
needed.

Even in subsystems where the original goal may have been to support the needs
of Perl6, many of those were designed and developed before people knew too
well what Perl6 needed. There was a lot of guessing, and a lot of attempts
made to become some sort of hypothetical, language-neutral platform that would
some how end up supporting Perl6 well, without ever taking the needs of Perl6
into account specifically. It's like throwing the deck of cards in the air and
hoping that they all land in an orderly stack. It's almost unbelievable, and
thoroughly disappointing to think that the "Perl6 VM" would do so little over
time to address the requirements of Perl6 and keep up with its needs as the
understanding of those needs became more refined.

Around the 1.0 release Perl6 moved to a new separate repository which
severely decreased bandwidth in the feedback loop. Whether this was a good
move in terms of increased project autonomy, or a bad move in terms of
decreased collaboration is a matter I won't comment on here. After the two
projects separated, Parrot added a deprecation policy which guaranteed that
our software couldn't be updated to reflect the maturing Perl6 project as it
gained steam.

The short version goes like this: Parrot was supposed to be the VM to run the
new Perl6 language. At few, if any, points in the project history did Parrot
focus strongly on the needs of Perl6. Now, a decade later, people act shocked
when Parrot doesn't meet the needs of Perl6 and meet them well. This, I
believe, is the root of the problem.

Look at other VMs like the JVM and the .NET CLR. The JVM was developed with a
strong focus on a single programming language: Java. When the JVM became
awesome at running Java, and became a great platform in its own right, other
languages like Clojure, Groovy and Scala started to pop up to take advantage.
This is also not to mention the ported versions of existing languages that
also found a home there: Jython and JRuby are great examples of that. The .NET
CLR was set up with a focus on the languages C++, C# and VisualBasic.NET.
Once the CLR became great at running these, other languages started to
pop up: F#, IronPython, IronRuby, and others.

Those other VMs became great because they picked some languages to focus on,
did their damndest to make a great platform for those, and then were able to
leverage their abilities and performance to run other languages as well. Sure,
we can always make the argument that Scala and Groovy are second-class
citizens on the JVM, but that doesn't change the fact that both of those two
run better on JVM, even as second-class citizens, than Perl6 runs on Parrot.

Somewhere along the line, the wrong decision was made with respect to the
direction Parrot should take, and the motivations that should shape that
direction. We need, in this time of introspection and reorganization, to
unmake those mistakes and try to salvage as much of the wreckage as possible.

It should be obvious in hindsight where mistakes were made, and how we ended
up in the unenviable situation we are in now. This isn't to say that the
people who made those decisions should have known any better. At the time
there were good-sounding reasons all around for why we needed to do certain
things in certain ways. Hindsight is always clearer than foresight. It's easy
to say that Rakudo is to blame because Parrot is filled with half-baked code
that should have been good enough for Perl6 but never was, and then a
deprecation policy that Rakudo expects to be followed. It's easy to
misattribute blame. *I understand that*. What we shouldn't do is keep
following that line of logic when we know it's not true. It's not correct and
we need to get passed it. Set it aside. Put it down. Walk away from it.

Look at the example of 6model. I don't know what the motivations were behind
the design and implementation of Parrot's current object model, but I have to
believe that it was intended to either support or enable support through
extensibility of the Perl6 object model. It failed on both points, and
eventually Rakudo needed to implement its own outside the Parrot repo. It's an
extension, which means it works just fine with Parrot's existing systems and
doesn't cause undue interference or conflicts. 6model is far superior to what
Parrot provides now, and is superior for all the languages that we've
seriously considered in recent months: Cardinal (the Ruby port) was stalled
because it needed features only 6model provided. Puffin (the Python port)
needed 6model. Jaesop, my new JavaScript port, is going to require 6model
because the current object model doesn't work for it. These represent some of
the most important and popular dynamic languages of the moment, and all of
these would prefer 6model over the current Parrot object model by a wide
margin. So ask yourself *why the Rakudo folks were forced to develop 6model
elsewhere, and why Parrot hasn't been able to port it into its core yet*. Ask
yourself that, and see if you can come up with any reasonable answer. I can't
find one, other than "oops, we screwed up BIG".

6model should have been developed directly in the Parrot repo. Everybody knew
that our object model is garbage and that 6model was going to be a vast
improvement. Maybe we didn't know how much it would improve, but we knew that
the amount would be *any*. But instead we had a policy that effectively
prevented that kind of experimentation and development, and a culture that
claimed doing things the Perl6 way would prevent us from attracting developers
from other language camps. Again, despite the fact that the developers of
other languages on Parrot desperately wanted an improved object model like
6model, we basically made it impossible.

And then because of that mistake that was made, if we finally want to get the
real thing moved into Parrot core where it belongs, we have to spend some
significant developer effort to do it. Of course Rakudo already has 6model
working well where it is, so moving 6model into Parrot core is listed as
"low priority" by the Rakudo folks. Not that we can't do it still (and we
will do it), but why would they prioritize moving around something they
already have? We shot ourselves in the foot, but the bullet ricocheted a few
times and hit us in the other foot, the hand, and then shoulder.

There's a sentiment in the Rakudo project that the fastest way to prototype
a new feature is to do it in NQP or or in Rakudo and eventually maybe backport
those things to Parrot. That's a devastating viewpoint as far as Parrot devs
should be concerned, and we need to do everything we can to change that
perception. It's extremely stupid and self-defeating for us to hold on to
ugly, early prototypes of Perl6 systems and not jump at the ability to upgrade
them to the newer versions of the same Perl6-inspired systems where possible.
This is especially true when there are no other compelling alternatives
available, or even clear designs for alternatives. It's also extremely stupid
of us to make it harder for people to improve code that have frequently been
referred to as "garbage".

There is nothing in Parrot that is so well done that if we were asked by our
biggest user to change it that we shouldn't take those suggestions seriously.
In most cases, we should take those suggestions as command. We're VM
people and maybe sometimes we might know better (or think we know better) or
at least think about things differently. We can have the final say, but we
should take every suggestion or request extremely seriously. Especially when
those requests come from active HLL developers and *Especially* when those
developers are part of a project like Rakudo.

We should be much more aggressive about moving our object model to 6model. We
should be very aggressive about moving other Rakudo improvements, in whole or
in part, into Parrot core. Things like the changes required by the argument
binder, or the new multi-dispatcher. We should also be very aggressive about
having other such radical improvements prototyped directly in Parrot core,
especially where we don't have an existing version, or where our version is
manifestly inferior. Parrot core is where those kinds of things belong. Of
course, we need to keep an eye towards other languages and make tweaks as
appropriate, but we need to pursue these opportunities when they are
presented.

dukeleto sent [an email][duke_email] to the parrot-dev list in follow-up,
trying to lay out some ideas for a new direction and a new vision for Parrot.
Some of his ideas are good, but some need refinement. For instance, he says
that a good goal for us would be to get working JavaScript and Python
compilers running on Parrot and demonstrate painless interoperability between
them. I do agree that this is a great goal and could bring Parrot some
much-needed attention. However, it can't be our only goal.

[duke_email]: http://lists.parrot.org/pipermail/parrot-dev/2011-September/006192.html

Right now Parrot has one major user: Rakudo. There's no way around it, in
terms of raw number of developers and end users, they are the single biggest
user by a mile. No question. For us to put together a vision or a long-term
roadmap that doesn't feature them, and feature them prominently, is a mistake.
There may come a time in the future when they decide to target a different
VM instead of Parrot. That time might even be soon, I don't know and I can't
speak for them. What I do know is that so long as Rakudo is a user of Parrot,
Parrot needs to do its damndest to be a good platform for Rakudo. A better
vision for the future would be something like: "Make Parrot the best platform
possible for Rakudo, but do so in a way that adequately supports and does not
actively preclude implementations of JavaScript and Python". Talk about having
a vision with sufficient focus!

I'm also not taking a jab at any other languages. Ruby, the Lisps, PHP and
whatever other languages people like can be added to the list as well.
JavaScript and Python are the two dukeleto mentioned and are two that I happen
to think are as important as any of them, and are good candidates to be the
ones we focus on. I would love to have a Ruby compiler on Parrot, and many
others as well if people want to work on them.

If we increase performance by something like 50% and add a bunch of new
features and Rakudo *still* leaves for greener pastures, at least we are that
50% faster and have all those new features. It's not like making Parrot better
for Perl6 somehow makes it instantly worse for other languages. Sure we are
going to come to some decisions where moving in one direction helps some and
hurts others, but the biggest things on our roadmap right now don't require
those kinds of hard decisions to be made. There is plenty of work to be done
that brings shared benefit.

We want JavaScript. I know, because I'm working on it personally. We want
Python too. Right now, we have Perl6 and we should want to keep it. We should
want to do it as best as we can. Talk that involves distancing the two
projects, or even severing the link between them is wrong and needs to stop.
Saying things like "Well, the python people aren't going to like a VM that is
too tied to Perl" is self-defeating. We don't have a working, complete
Python compiler on Parrot and havent been able to put one together in a
decade of Parrot development. We do, however, have a damn fine Perl6 compiler.
If Puffin, a product of this summer's GSoC, continues to develop and becomes
generally usable and more mature, the conversation changes. If Jaesop, my new
and extremely immature JavaScript compiler comes around, the conversation
changes. But until we have those things, we do have Perl6 and that needs to be
something we focus on.

There are two directions we can logically go in the future: We can do one
thing great, or we can do many things well. We can focus on Perl6 and be the
best damn Perl6 VM that can possibly be, or we can improve our game to support
multiple dynamic languages, but not be the best with any. Both of these are
fine goals, and I suspect what we want to do eventually lies somewhere in the
middle. We do know what path JVM and CLR took, and where that got them. We
also know what path Parrot has pursued for a decade, and where we are now
because of it. I think the course of action should be very clear by now: So
long as Perl6 is our biggest user, it needs to be our biggest source of
motivation. Parrot is not so strong by itself that it can afford
to ignore Rakudo or become more separate from it than it already is. Parrot
might be that strong and mature one day, but that day isn't today.

In direct, concrete language, this is what I propose: We need to focus as much
effort as we can to be a better VM for Rakudo Perl6, including moving as much
custom code from the NQP and Rakudo repos as possible into Parrot core to
lower barriers and increase integration. We need to do that, trying to put
priority on those parts of the code that are going to affect JavaScript and
Python implementations and making the difficult decisions in those cases. When
compilers for those languages become more mature and we start to run into
larger discrepancies between them, we can start revisiting some decisions as
necessary. Until then, Rakudo is our biggest user and beyond that they
are friends and community members. We need to focus on their needs. We need to
focus on making Rakudo better, and we need to focus on making Parrot better
for Rakudo. Everything else will come from that, if we do our job well enough.


