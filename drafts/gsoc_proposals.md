The proposal deadline for GSoC 2011 has come and gone. All the proposals that
we are going to receive have been received, and anybody who missed out on
the deadline are just going to have to wait until next year. Parrot has
received a grand total of 15 proposals from 14 individual students, some of
which are extremely impressive.

However, by this stage in the program I find that I'm already exhausted from
it. I've spent the last few days reading proposals and giving feedback, in
many cases the same exact feedback, to all of the students. There are some
issues and mistakes that every single student has made. Other issues that
were extremely common. There are a few issues that seem to be common to the
batch of last-minute proposals we received in the hours and minutes before
the deadline. In this post I want to talk about these various problems and
issues, so that we have something written down for next year. Obviously,
nothing I write today is going to help this years students, but it's only
after going through the application process as the organization admin for
Parrot that I've been able to distill some of these conclusions and opinions.

## The Code Is Only a Small Part

The single biggest, most common criticism of the proposals I've
received--every single one of them--was a complete omission of documentation
and testing. I could go off on a tangent and start complaining that schools
aren't teaching these valuable aspects as well as they should be, but that
sort of misses the mark. Many of our students aren't in CS programs, so we
wouldn't expect best practices related to coding to be taught in other
subjects.

Regardless, that's academia and this is the real world. Out here, in the
bright light we do a lot more than writing and rewriting merge sort
implementations. GSoC is all about taking your coding skills up to the next
level, and a big part of that is by understanding the whole software process.

Working software is one thing. Properly documented and tested software is
something else entirely. Documentation and tests are as important as the
working software. Tests prove that the software works as expected and that it
doesn't break when things change in the dependency chain. Documention shows
how to use the software, and maybe even how to modify it.

For your GSoC proposal, you should be explicit about your documentation and
tests. Mention what documentation tools you're going to use. Mention what
test tools you're going to use. For goodness sakes, don't try to homebrew
either. Use existing toolchains where possible, and try to pick ones which are
standard for your problem space. For instance, if you're running on .NET, you
could pick NUnit for your tests. It would not make a hell of a lot of sense
to use Test::More from the Perl5 library to do it. If you're writing Python
code, maybe you want to use doxygen for your documentation. You probably don't
want to use javadoc.

## Get Involved

I say this all the time, yet so many students seem to take it as a suggestion.
When we are evaluating proposals, we do so with an eye towards completability.
Can you actually complete the work you propose in the time alotted? There's
no way we can predict the future, but we can look for some indicators to help
us make better predictions.

We ask students to discuss their backgrounds on their proposals, but there is
only so much you can learn from this. Most students don't have huge online
portfolios so early in their careers for us to look at. Some of them have
Github accounts we can look at, but most don't.

The single best way that we can get an understanding of your capabilities as
a student is to interact with you personally. You *must* come chat with us.
I don't care if it's over personal email, or on a mailing list, or on IRC.
One way or another you *must* get in touch with us. This isn't optional. We
**will not** accept an application from a student we haven't spoken to,
regardless of the quality of the proposal.

But here's the funny thing. Talking to us is a great way to get feedback on
a draft of your proposal. The students who have been working with us the
longest, some of whom appeared in our chatroom the day it was announced Parrot
would be participating in GSoC, have the best proposals. This is because they
have had all the extra time to chat with us, find out what kinds of projects
we want, and get feedback from us about how to make those proposals better.

On a side note, the same thing goes for GSoC mentors: If we don't know who you
are, you aren't going to be a mentor for us. There's no negotiation on this
subject. Being a mentor involves non-trivial responsibilities for both the
student and the Parrot Foundation. This isn't the kind of thing we entrust
to just anybody, and it would be a disservice to everybody involved if we did
otherwise.

## Timeline

The timeline is the single most important part of the proposal. The timeline
serves several purposes:

1. It shows that you have a plan
2. It shows that you understand the work involved, and are able to estimate
   it.
3. It shows that you are familiar enough with the task to break it down into
   reasonable chunks
4. It shows that you are working on short timelines and small milestones,
   which are easiler to accomplish than a single large deliverable at the end
   of the summer.
5. It gives us an easy way to keep track of your progress, and evaluate how
   well the program is going for you.

Other organizations might be different, but here is what we want in a
timeline:

1. Break it up by weeks at least (smaller time periods are fine too, if you
   want to provide that level of detail).
2. Each week should have specific, explicit milestones. You should be able to
   say what you are going to be delivering and able to demonstrate at the end
   of each week.
3. You should include documentation, test, and build infrastructure milestones
   in your timeline. **Do not save these things until the end**.
4. You should include a prioritized list of additional items that you will
   work on if you are ahead of schedule.
5. You should include a prioritized list of items which you can reasonably
   cut if you are running behind schedule (This **DOES NOT** include
   documentation and testing).

Timeline entries like "Week 3: Keep working on Foo" are not specific enough.
If you have continuations like that in your timeline, it means you need to
break your big tasks down into smaller tasks. If you have an entry like
"Week 5: Fix bugs", that means you are planning to have bugs, or you are not
planning to be fixing them as you go, or you aren't doing testing early
enough to find the bugs earlier. Create mechanisms to help find, diagnose,
and fix bugs earler, and don't devote large blocks of time to it in your
proposal. This is why testing is so important. Testing helps you find bugs
before every commit, long before they pile up and require an entire week to
fix.

## Documentation And Testing

Seriously. I can't reiterate this enough. Learn it. Live it. Love it.

Documentation and testing are extremely important. They should become integral
parts of your daily coding sequence. Some people do test-driven development,
where you write a test first, then write the code to satisfy the test. Write
a test, write the code. Write a test, write the code. Continue.

Some people do it the other way. Write code, then write a test to prove it
works. Write the code, write a test. Repeat.

Some people get documentation into the mix as well. Each week, write draft
documentation for the features you are going to work on that week. This makes
sure you have an idea in mind and have planned how it is going to work. Then,
in short iterations, write your code and tests throughout the week. At the
end of the week, review the documentation to make sure it's still accurate,
changing whatever needs to be changed to make it accurate.

Documentation and testing are not just add-ons, they should be an integral
component of your development process. These things are tools, and if you
use them properly, they can help make your work either.

If you don't understand how to properly use testing in your application, or
how to properly leverage documentation, ask. That's what mentors are for.

## Write, Edit, Revise

I learned this sequence back when I could still count up to my age on my
fingers. Proposals are important, because if we don't get as many slots as
we get proposals, we have to start making decisions. Who gets accepted, and
who gets rejected? Part of the answer is going to be found in the strength of
your proposal.

This is why it's so crucial that you start interacting with us long before
the deadline. Send us drafts, and we will send you feedback. Edit, then
resend. Repeat. The best proposals we have are on draft 3 or 4 or 5. They
always get better over time. Always.

If we have to make a tough decision between two projects, one of which is a
lousy proposal and the other one is a great proposal which we've been giving
feedback to over a period of several days, the choice becomes pretty clear.

## Pick a Good Project

This one should be obvious, but judging by some of the proposals we've
received it may not be. You need to pick a good project. "Good" in this
sense means a few different things. First, it needs to be beneficial to
Parrot. We won't agree to mentor and fund just any project, it needs to be
good for us. Don't propose a project written in a language that doesn't run
on Parrot. Don't propose a project which isn't useful to the Parrot community
at large. Don't propose to write software we don't want, using toolchains we
don't support, or to benefit people who aren't us. Don't do that.

If you put in a proposal that violates any of these maxims, it becomes
pretty clear pretty quickly that you aren't familar enough with our project.

Of course, Parrot isn't the only party here. You need to pick something that's
going to be useful and interesting to you. This has to be something that
you're going to get excited about, because you're going to be spending several
hours working on it every day.

Looking at a list of ideas on a webpage is nice. You can find an idea that
interests you, and base a proposal around that. However, this shouldn't be
your only option. Feel free to propose something that's not on the list. Feel
free to take the kernel of an idea from the list and turn it into something
that works for you.

Don't just pick the easiest thing on the list and expect to breeze through
GSoC. That doesn't do anybody any good.

Most importantly, once you have an idea in mind, *talk to us about it*.
There's no escaping it, at some point we are going to need to chat. Talk to
us about your idea to get feedback about it. We can help you pick a good
direction to follow, we can help you fill in some of the important details,
and we can make sure you find a project that is going to maximize benefit
to both parties.

Of the students who came to us early with ideas, not a single one of them has
kept their ideas unmodified. In all cases, ideas have gotten *better* with
feedback and revision, and better ideas are more attractive to us.

## Next Year

So there's a short treatise about making successful GSoC proposals. I hope it
can help some students next year.

Even though the application deadline is passed, were not without hope quite
yet. Now we're in a period of feedback and interaction where we have to pick
successful applications from our list of submissions. If you  have submitted
a proposal late and didn't take some or all of this advice, you still have
time to do it. Don't wait! The clock is ticking, and we have only a very
limited amount of time to do everything I say above. Take this seriously
and treat it like a job. It's important and the benefits are real.
