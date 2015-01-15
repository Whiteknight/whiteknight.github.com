---
layout: post
categories: [Parrot]
title: Why I Stopped Working On Parrot
---

For the sake of just wrapping up this subject, here is the post about the 
technical side of Parrot that I promised. I'm sorry that it's so large but I
have a lot to say and I hope that in saying it I'll provide some kind of
benefit to somebody.

Also, everything that I say here concerns Parrot of 2-3 years ago when I was
still working on it. I know other people have continued to develop it so some
of the problems I mention might already be resolved. This post is **absolutely
not** intended to demoralize the people currently working on Parrot or future
contributors who might be interested in joining. All I can talk about is what I
was thinking and feeling about the project, all of which is subjective and
possibly incomplete or mistaken. In all this I want to make clear that I stopped
working on Parrot because of my own personal goals and needs, which are almost
certainly not the same for other people.

## The Dream of Parrot

There was a great dream which was Parrot. It was going to be many things, some 
of them great and visionary while some of them were a little more pedestrian 
(but nonetheless necessary). At the most basic level, the Perl world wanted a VM 
to run Perl (some version of it) with proper abstraction boundaries and without 
the kinds of twisted coupling nightmares that plague the current implementation 
of Perl5. From the very outset the Perl6 language project aimed to resolve 
exactly these kinds of problems: Start with a specification first, and design it 
in such a way that the execution engine could be divorced from the compiler, and 
multiple implementations could exist side-by-side.

Python and Ruby, as two immediate examples, have benefited strongly from this 
kind of arrangement. Yes there are default implementations of these languages, 
but lessons learned during development of competing VMs and runtimes helped to 
strengthen the specifications and the community, increase competition, and 
expand the number of platforms where these languages can run. They are better 
for it.

Parrot, along this line of thinking, was supposed to be the *primus inter 
pares*. Yes, Perl6 should be able to run anywhere and be implemented on a number 
of VMs and platforms. However Parrot would be the only one among these that 
would be tailor-made to fit Perl6 perfectly. Perl6 can run anywhere, but if you 
want *the best* Perl6 experience, with the most cutting-edge features, Parrot 
should have been the go-to platform.

While this was among the earliest of design goals, it very quickly fell along 
the wayside as something that the Parrot developers didn't think was necessary. 
Let's not separate "me" from "they" here, I definitely believed the same thing 
for a while, but I wasn't really around when these decisions were being made.
You can't blame the early Parrot devs for falling into the trap that they did. 
Once some code starting being written and the two projects became separate, the 
Parrot folks gained some ambition. We didn't just want to have a VM that runs 
Perl6, because many of the features that Perl6 needs are also baseline 
requirements for other dynamic languages as well. Wouldn't it be great if 
Parrot supported many languages and enabled interoperability between them, in
the same way that .NET was doing with VB and C#, or how Java was starting to
with Scala and other various language ports? Sure, you can say that .NET only
really had eyes for C# despite begrudging support for other languages, or that
Scala was always a bit of a second-class citizen and oddity on the Java
platform, but these things existed and there was real, demonstrable 
interoperability at play.

Despite static kinds of languages being able to work together for years
(depending on the compilers used and the linking options provided, of course), 
dynamic languages were all self-contained unto themselves and didn't really have
any of these benefits. With Parrot, no longer would every language need to
provide it's own standard library (especially in cases, such as PHP, where the 
default standard library is severely lacking, in design if not in 
functionality). Parrot could provide a huge common runtime, and every other
language could share it directly, or write some thin wrappers at most.

There was also the idea of the rising tide that lifts all boats. In a world 
where every dynamic language has it's own VM and own runtime, improvements in 
the fundamental building blocks of these need to be separately reproduced for 
each. Develop a better garbage collection strategy? Implement it a dozen times 
or more, in each platform that wants it. Develop a better JIT algorithm? 
Implement it a dozen times or more for each platform that wants it. Better 
unicode support? Better threading and multitasking? Better networking? Better 
object model? Better native call or even native types? Better optimizations? 
Better parsers and parser-building tools? For every one of these things, every 
time something new and awesome is developed, we can implement it dozens or even 
hundreds of times. And if we don't implement each new thing on each old 
platform, the divide between them increases.

Take a list of all the haves and the have-nots. Java has great unicode support. 
Ruby has a great object model. V8 has great JIT. Python has great green threads 
and tasklets. PHP has great built-in bindings to databases and webservers. .NET 
has some great optimizations. Perl has CPAN. JavaScript needs a better object 
model. Python needs better threading and multitasking. PHP needs unicode 
support, Perl needs JIT and optimizations, and the list goes on. One of the 
goals behind Parrot was that we could bring together all the strengths into one 
reusable bundle and eliminate the most common weaknesses, and only need to do 
it once.

To recap, here are the three big goals of Parrot, as they have been communicated 
over time (though they haven't always been equal priority):

1. To be an initial "best fit" VM for Perl6
2. To be a runtime for multiple dynamic languages where interoperability is 
   possible
3. To be a single point of improvement where costly additions could be made 
   once, and many communities could benefit from them together.
   
I left the Parrot project for a simple reason: The goals that I had with regards 
to Parrot were met and the things that I thought could be accomplished were 
accomplished. Just, not with Parrot. Other modern VMs, whether by accident or 
design, have achieved the kinds of goals that were supposed to set Parrot apart,
and all the while Parrot was not progressing hardly at all. For years it seemed
like we needed to take two steps backwards first, before attempting any step 
forward. The system just wasn't very good, and no matter how much we worked to 
improve it, it felt like we were being weighed down by an unimaginable burden of
cruft and backwards compatibility. The whole time we were slogging through the 
mess, other VMs were surging forward. In the end, I got the VM that I had always
wanted, and it was called .NET (Java and V8 are also great, but I don't use
either of them nearly as much). Let's look at those three goals again
to see why.

We can make arguments all day long that .NET or JVM don't have an object model 
or dynamic invoke mechanism which is 100% exactly what is needed by Perl6, 
Python, Ruby or PHP. "There's always going to be friction", I've said it and 
I've heard others say it around me. Yes, this is true that these two big VMs 
will never cater to P6 on bended knee. However, every system has trade-offs and 
the small amount of friction is outweighed by the other benefits: large 
libraries and library ecosystems, large existing user bases, near universal 
desktop penetration (for .NET, it's near universal on Windows systems, but with
Mono, Xamarin and new OSS overtures from Microsoft this situation is improving
and rapidly) and significant footprint in the ever-growing mobile world. And
further, because .NET and JVM have better memory models, garbage collection, JIT
and various other optimizations and performance enhancers, the performance 
of P6 on those platforms will likely be just as good if not better than 
performance on Parrot for the forseeable future. Parrot not only has to provide 
less friction (which it doesn't even do), but it also needs to have comparable 
performance and memory usage, which it simply does not.

Interoperability is supposed to be an area where Parrot excelled beyond the 
norm, but as of 2012 it did not work as expected and *I don't even know why it 
didn't work as expected*. People who claimed to know about it said it wasn't 
working, and there was a ticket open somewhere for "Make interoperability 
happen", but it didn't work right and nobody was trying to fix it. When I asked
what needed to happen to get it working, I could never get an answer besides "it
just doesn't". 

Compare to a platform like .NET where you can write and interoperate all the 
following languages: C#, VB.NET, C++, F#, IronPython, IronRuby, JScript (and 
IronJS), and various dialects of Lisp, Clojure, Prolog, PHP, Ada, and even 
Perl6. Yes, you read that correctly. As of the time I left you could, perhaps 
with some effort, write a Perl6 module on .NET using Niecza and interoperate it 
at some level with a library written in C#. Maybe Niecza has lost functionality
in the past few years or maybe that project has since been abandoned, but last
time I looked Niecza on .NET was miles ahead of Perl6 on Parrot in terms of
language interoperablity.

Don't even get me started on the various compiler projects which translate 
various dynamic languages into JavaScript for use in the browser. For all it's 
flaws, JavaScript is indeed turning itself into an "assembly language for the 
internet". You can, today, compile all sorts of languages into JavaScript, load
them and run them together in a browser. The JavaScript environment even has 
it's own trendy new languages which don't exist anywhere else (CoffeeScript, 
etc). When you consider the amazingly productive performance arms race between 
Microsoft IE, Google Chrome and Mozilla FireFox (among others!), it's easy to 
see why the platform has become so attractive. Throw Node.js into the mix and 
suddenly JavaScript starts to look like just as compelling a platform, and more 
versatile than some of the desktop-only options in .NET and JVM. There's always
going to be a little bit of friction translating any language to JS with its 
goody object model, but a smart developer is going to take a look at all the 
benefits, and do a simple calculation to see if it's still a worthwhile 
platform. Many people will decide that it is.

## Performance and Features

I hear people asking questions like "Is Parrot fast enough?" Which hasn't been
the question to ask, really. Parrot doesn't provide the features it is supposed
to, so it doesn't matter, in my mind, if it does the wrong things quickly
enough. Sure, Parrot has been dirt slow (I hear it has since gotten faster) in
part because we were doing too many things that we didn't need to do and we
weren't doing enough of what we needed. So a language like Perl6 needs to either
suffer through the bloat of our method dispatcher, or else write their own to
do what they actually need. Guess what they did?

In terms of having an awesome feature set on which all languages can leverage, 
and a single point for making improvements where all language can benefit, 
Parrot is a mixed bag. There are some places where I believe that Parrot really 
does provide awesome features. The hybrid Parrot threading model is, while
incomplete at my last viewing, among the best designs for a built-in threading 
system that I've seen since. But then again, when you look at the new Futures 
and Promises features built-in to C# 5, or the `java.util.concurrent.Future` 
library in Java, or when you look at the event-based everything in Node.js, the 
Parrot offering doesn't stand out as much. It's one great design among a pool of
other, similar, great designs. Parrot's native call system is conceptually among 
the best, though probably is edged out by some of the other options. Parrot's 
unicode support and string handling in general are pretty good. Could be better, 
but still pretty good (lightyears ahead of some of the competition. PHP comes to
mind). 

Where Parrot was lacking was in everything else. The object model, 
especially, stood out as a place where the fail was particularly strong. Parrot 
doesn't have JIT (and what it used to have was a dumb bytecode translator which 
only worked on x86 (which wasn't even the most popular platform among our
developers) and helped propagate the misconception that we had a *real JIT*, 
which we never did). Our calling conventions subsystem was poor, but not because 
the implementation was bad. For what it was supposed to be, the implementation 
was actually decent. The problem is that the *specification* was bloated and 
painful, and the abstraction boundaries were drawn in the wrong place. Every 
call had to create and then decode a CallContext object, but Parrot did all of 
this internally. This meant that Parrot took responsibility for every type of 
passable argument, including named and optional parameters, and made it 
unnecessarily difficult for languages which didn't need exactly these features 
implemented exactly this way to do anything different.

For the record, calling conventions were a huge part of the reason why my MATLAB 
clone, Matrixy, died. Because we never had easy access to our own CallContext 
object, we were never able to properly implement some of the basic features 
(like variadic parameter and result lists) which were required by even the most 
basic subsets of the standard library. NQP and Rakudo had to go to extremes to 
write their own argument binder for making calls, effectively cutting the bulk 
of the Parrot code out of the loop. 

My JavaScript port, Jaesop, died because of object model problems. More than 50%
of the code written for that project was trying to shoehorn the JavaScript 
object model into the Parrot one, and barely got even the basics correct. Maybe
this is because of some fundamental misunderstanding on my part, I'm not a JS
expert and maybe I was missing some kind of crucial Eureka moment in the design
of it. Regardless, I was having a hell of a time fighting with the object model
to try and get the result I wanted, and a better object model would have let
even my poorest designs work. The object model is also a huge reason why Python 
and Ruby projects floundered and died too. People wanted all these languages to
run on Parrot, and the Object Model was the single biggest reason why nobody
could make it work.

My libblas bindings, PLA, was plagued by the same problems. The object model 
basically required PLA to be written in C, and performance suffered because of 
the twisted calling convention problems. My database bindings, ParrotStore, had
the same limitations. As I was developing these, The P6 folks were developing 
their own bindings which used the (much nicer) 6model and the (much nicer) P6 
native bindings instead. After a while I had to ask myself why I was fighting in
the weeds so much, when the P6 people were rising above the problems of Parrot 
and doing things better? If my writing these things wasn't helping anybody, why
bother with it?

Notice that the P6 folks were having their biggest successes when they bypassed
Parrot, which isn't exactly a roadmap for synergy and mutual success. One day,
and I saw it coming like a freight train, P6 was going to realize that they 
could have the most success by bypassing Parrot entirely. I was not at all
surprised when I started seeing blog posts in my daily feed about MoarVM.

Rosella was actually my one project which didn't run into too many Parrot
problems. But then again, I wrote much of it to work around Parrot issues that
I was aware of because of my knowledge of the internals. Somebody besides myself
trying to write a similar project would have been in big trouble.

The goal of all these projects I worked on was to provide a substrate of common 
functionality that other people could build on top of. If many languages can be
translated into JavaScript, and if Parrot has a JavaScript compiler, we start to
gain language adoption and interoperability for free. If parrot has an 
attractive and full standard library, people will be able to build on top of 
that to make bigger things, faster. If we have good infrastructure like unit 
tests, project templates and build tools, people will be able to leverage them
to get new projects from conception to production faster. This just isn't the 
way things worked out. The tide was indeed rising, albeit slowly and uncertainly,
but all of the ships had already set sail.

## Project Leadership

Allison was a pretty great architect before she reached her own burnout point.
When she made her absence official I asked for the Job of architect in her
stead. The job instead went to cotto which was probably the right choice at the
time. While I had plenty of free time and energy to devote to the role, I was
young, immature, ignorant of some of the big ideas, inexperienced in leadership
and software architecture, and abbrasive to talk to sometimes. Having time and
energy, while an architect certainly needs these things, wasn't enough reason
for me to be it. 

We know now in hindsight that cotto didn't really have the free time to keep up
with the position either. I don't know exactly what was eating up his time but
I can guess. Following the economic meltdowns in 2008 and 2009 many of our
best developers were spending more time at work, fighting to keep jobs that were
melting away, or being forced to pick up slack for other jobs that no longer
had people. When you're feeling a little pesimistic about Parrot, and your work
life is taking more time and generating more stress, your open source project
participation suffers. I don't know exactly why cotto left, though I assume this
and burnout and pessimism about the project all played their own parts in it.

I was trying to do design work and rewrite old specs and make big changes, but
without an architect there to take the thirty thousand foot view and sign off
on things, I feel like I got caught in a bit of a rut. We had an architect for a
reason and I respected the position enough to not go outside of that. But when
you go for so long with Allison not participating and then she hands the job to
cotto and he isn't able to put in enough hours, I feel like a lot of the things
I wanted to accomplish were stalled.

What I can say is that if I were architect I would have kept things moving a 
little longer, though with my own burnout fast approaching and my inexperience
and other problems in play, who knows if I would have moved us to a place we
wanted Parrot to be.

## Perl6

I've talked about P6 quite a lot because P6 was really the central player in all
this. Without it, Parrot would have been nothing and would have had no purpose
for existing at all. Parrot made many mistakes with respect to Perl6: 

1) Not treating it like the Most Valuable Project
2) Kicking it out of the Parrot repo and forcing it to become a separate, 
   stand-alone project.
3) Not catering to the needs of Perl6 more closely
4) Acting like the needs of any other language were important at all, much
   less as important as the needs of Perl6.

And again, I understand why people did it. They wanted Parrot to be language
agnostic and they wanted this utopian dreamland of language interoperability.
The problem is that you need two languages running on your VM to worry about
interoperability, and we only had the one. And then we kicked it out of the nest
to make room for the other projects that weren't coming.

Before I left I was trying to refocus the project to be more of "The Perl6 VM"
and less of "The VM that hosts many languages and, oh yeah, Perl6 but not well"

I wanted to merge 6model into Parrot core and I wanted to make some major
changes to the method dispatcher to more closely mirror the model P6 was using
(which is, as I have known for a long time, much closer to the "right" way to do
it). 

Here's the part of the confession that should be revelatory, because I've never
expressed these thoughts publically before: I was really starting to dislike
Perl6. I was starting to feel that (a) it would ever be completed and (b)
that if it was completed it might not be any good. Development on Perl6 has 
taken a very long time, much longer than development on other languages or 
compilers. In defense you might say "But Rakudo has spent years fighting with
problems in Parrot, it would be far ahead of where it is now were it not for all 
those lost years". I'll agree with that to a point. Rakudo certainly did lose
time with Parrot, but even allowing for 5 years of purely lost time, it has still
had a huge development cycle *and nobody is calling it complete yet*.

Plus, it's not like Parrot has been the only host in town. Rakudo has a JVM 
backend last I heard, and there's Niecza on .NET, neither one of which has the
problems that Parrot has. Despite these things rendering the problems of Parrot
moot, Perl 6 development hasn't exactly accelerated forward.

People say "oh but those VMs aren't designed for dynamic languages! There's 
extra friction!" Which is true to a point, but languages that run on those VMs
or have been ported to them don't seem to mind. Both JVM and .NET currently
host fully operational versions of JavaScript, Ruby, Python, and PHP, and you 
don't hear those communities complaining about how impossible it is to make
compilers because of the inherent friction.

In theory Parrot should have been able to do a little better, but .NET and JVM
aren't exactly unusable for the purpose. And when P6 runs on those platforms,
you can't complain that Parrot is the anchor holding your whole operation back.
When you're running on a platform as stable and usable as JVM, for example, and
you still are spending year after year on development just to get to a "yes,
it's done and ready" v1.0, maybe the problem isn't with the underlying platform.

So that leads me to a major existential problem: If Parrot should be targetted
squarely at Perl6 (and, for any chance of success, it *should be*) and if I 
really don't like Perl6 and don't believe that it will do what it promises (and,
I don't) then it's hard to log in every day and spend hours and hours working on
Parrot.

We could have retargetted Parrot *again* to not focus on Perl6 and start working
on those other languages that people wanted (Ruby, Python and JavaScript would 
have been the best contenders) but then we would have gone from one active
downstream project to none, and that would have been instant death for the
project. Out of the fry pan, into the fire.

I think there's hope for Perl6, and I sincerely wish that language well. 
They aren't going to see any kind of adoption until they are willing to put a 
"complete and ready for production" sticker on the front of the box. If they are
unable to reach that point they need to reconsider their spec and their 
assumptions. If they are unwilling to reach that point, they need to take a long 
hard look inwards at the project culture.

## Why I left

I loved Parrot. I honestly did. I devoted years of my life to it, cleaning and 
coding and planning and designing and arguing and discussing. I spent hours of 
precious, limited free time hacking Parrot and trying to make it better. I
bought into the dream, and was doing everything that I could do to actualize it. 
Maybe we can take a certain amount of credit, that we had these dreams before
some of the other platforms which actually were able to reach them first. Maybe 
we played some small influential role in the development of other competing 
platforms, with people seeing what we were trying to do, deciding it was a great 
idea, and beating us to the finish line. Maybe these ideas were just common 
sense and other folks would have arrived at them without ever hearing about 
Parrot in the first place. I don't know how exactly all the pieces of the 
historical puzzle fit together, or who gets credit for what. What I do know is 
that we *did have the good ideas*, we just weren't able to implement them
correctly or quickly enough. We may not have won the race, but we were at least 
on the right racetrack. There's something to be said for that.

When I stepped away from Parrot, I thought that I just needed a bit of a 
breather. I was starting to feel the symptoms of burn-out, and I needed to step 
away and collect my thoughts. A few days turned into weeks. Weeks into months
and months over years. At some point I realized consciously that I had no 
intention of returning, and so I never did. 

I was burnt out over some of the big branches and features, but I was also 
getting down about the state of the foundation. I was down on Perl6 in general, and I was seeing them slowly but
surely moving to other platforms and leaving Parrot behind. I saw how hard it
was to implement any languages on Parrot, and I knew that, in this state, Parrot
would have no languages and be dead. 

The longer I was away the less I wanted to return. The things that I wanted to 
do for Parrot already existed, and instead of blazing a new trail, I would have 
been playing a frustrating game of catch-up, following in the footsteps of 
organizations like Microsoft and Oracle, Google and Mozilla, who each have much 
more than a few spare man-hours each week to devote to their projects.

In the end, when I added up all the reasons to leave and all the reasons to
stay, I decided my time with the project was over for good.

I'm not going to return to Parrot development. I don't harbour any regrets or 
ill-feelings, I just am not motivated to do the kind of work that needs to be 
done any more. There's work that I did in Parrot that I am, to this day,
extremely proud of. I didn't have any problems or quarrels with any of the
other developers, and I still count several of them among my list of friends.

I haven't joined any other open-source projects since I left Parrot, but I
have started looking for one that suits me. I'm not quite sure exactly what I'm
looking for, but I'll know it when I see it, and I'll devote as much of my time
as I can spare. 
