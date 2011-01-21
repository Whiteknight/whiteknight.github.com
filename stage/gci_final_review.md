---
layout: post
categories: [Parrot, GCI]
Title: Google Code-In, Final Review
---

The Google Code-In (GCI) program has been over for a little while, and I'm
finally ready to reflect on it a little bit. Frankly, the whole experience
was completely *exhausting*. The students were totally ravenous and ate up
tasks as quickly as we could create them. Every morning when I woke up I had
to review, test, and merge submissions. Every day I had to create as many new
tasks as I possibly could and answer questions about them. It was a lot of
work but extremely fun, extremely beneficial, and completely worthwhile.

We got to meet some extremely impressive young coders, many of whom I look
forward to working with again in other GCI programs, later in GSoC, and also
as regular open-source contributors.

When all was said and done, we had nearly 200 tasks completed by many
students. About a half-dozen students were working on Parrot tasks nearly
every day of the program. Our development velocity, in Agile-speak, went
completely through the roof. Now that GCI is over, we can clearly feel the
absence of these hard-working and extremely productive individuals.

I think now is a good time to reflect on the program, talk about what we've
learned, what we can do better next year, and what I think can change about
the program as whole.

In terms of the students, there is absolutely nothing bad to say. On average
they were all extremely bright, very productive, very competitive, nice,
friendly, and helpful.

Over the course of this program Parrot had offered a number of tasks for it's
students. In the beginning we had plenty of time to prepare an initial
selection, and there was much diversity in the first crop of tasks: We had
tasks involving code, testing, documentation, translations, user outreach, and
several other areas. When those all were completed, we started looking for
other areas that could be mined for new tasks. After all, the best play for
the Parrot Foundation was to provide as many tasks as possible, and try to get
as much "free" work out of the students as we could.

After the initial batch of tasks was emptied, we started looking elsewhere.
The first thing we did was offer several translation tasks to get our various
documents translated into several languages. What we ended up with was a huge
pile of translated documents of various levels of quality, but for which we
didn't even have an organizational infrastructure in place to manage. In fact,
thats a problem that the Parrot Foundation still hasn't solved: Where do we
put all these translated documents, how do we evaluate them, and how do we go
about keeping them up to date? At the time of writing this blog post, The
Parrot repository contains several branches which have translated documents
for which there are currently no plans to merge until we get this issue sorted
out.

We went through the ticket queue, several times, and turned any tasks that
appeared suitable into GCI tickets. Those went relatively quickly. These I
think were the most natural tasks: They were things that we already wanted,
and the tickets themselves provided a nice place for students to discuss their
results, post patches, receive feedback, etc. Parrot does have a very large
queue of outstanding tickets, but many of them are open precisely because they
are very difficult or even open-ended. While we still have hundreds of more
tickets, most of them are probably not suitable for a GCI task.

Later we started making several tasks to include code coverage of our test
suite. For some things, like the new Embedding API, this brough a huge influx
of valuable tests for the API which I am sincerely grateful for, but we also
got accused of "spamming" tasks and allowing certain students to inflate their
scores by rapidly completing tasks that were too easy. I'm happy to explain
why this is not the case, but it's still a little bit annoying to be accused
of it.

On a related note, improving our code coverage is a very nice thing to do and
can be a very tedious chore. It's a perfect thing for a contributor like a GCI
student to do because it is a task that is necessary but which rarely draws
sufficient attention from core developers. It's not a task that I personally
perform too often, for instance. On the other hand, having all these extra
helpers around can really help us get moving on core development tasks such as
fixing bugs, adding new features, cleaning/refactoring code, etc. We do want
improved code coverage, but we don't want to be feeding those kinds of tasks
to GCI students exclusively. We do need to keep a nice balance. Some students
really loved the coverage tasks, but others quickly tired of them and wanted
to work on other tasks.

So what have we learned this year in GCI?

The first lesson I've discovered is that the students will eat up code-related
tasks as quickly as we create them. They loved code. Their desire to write it
was insatiable. To be ready for GCI next year we should have at least 100
code-related tasks available up front, before the program even starts. We
should be prepared to write dozens more too. There were so many days during
GCI when we had no tasks left in the queue and the students were waiting
around for us to create more. That's an unacceptable waste of manpower and we
shouldn't let that happen again.

Translation tasks can be easy to do and are relatively popular with students,
but don't always yield great results. We have received several good
submissions of documentation translations, but we also received several which
were not good and almost could not become good after cycles of feedback and
resubmission. This can be a bit of a waste of time for the student and mentor
alike. Also, it's not always easy to find a developer to be a mentor or
proof-reader for translations in all languages. On at least one occasion one
of our developers had to call in a favor with a colleague to get a translation
proof-read. Those are not the kinds of favors we want to be expending without
good reason. Luckily for us the translation in question was high-quality and
didn't require much feedback or modification from the reviewer.

We received some readme submissions written in languages which we have not yet
been able to find a reviewer for, and may never find one. We as a community
have had to start a number of discussions about these kinds of things, and I
can't say with any confidence right now that we have a plan yet. Translations
of vital documents are nice things to have, but we as a community really
aren't prepared to receive such high volumes of them right now. Next year, I
do not expect us to offer so many translation-related tasks.

Other types of tasks such as documentation, research, and outreach tasks are
less popular. Luckily they are also much harder for us to come up with in
sufficient numbers. Students will do these if there are no other options, but
it's becoming clear that these kinds of tasks will sit in the queues while
students focus on the code. On a couple of occasions I felt like the quality
of these kinds of submissions was lower than expected, although I could have
been setting my expectations for them too high. Even if students loved these
tasks, there's no way we could create enough of them to keep up with demand.
We will want a handful of them for next year, but I don't suspect we will have
many or that students will request many.

Students have been very willing and even eager to jump into tasks for various
ecosystem projects such as [Parrot-Linear-Algebra][pla], [ParrotSharp][ps],
and [Winxed][]. Several new features (and accompanying tests) were added for
PLA. Several new features and an entire new test suite were added for
ParrotSharp. I find these things to be very valuable, and I would sincerely
hope that in future years the ecosystem projects represent a much higher
percentage of tasks for students to perform. These ecosystems represent a huge
variety in terms of tasks, technologies, and skillsets. I think students
do enjoy the variety.

[pla]: http://github.com/Whiteknight/parrot-linear-algebra
[ps]: http://github.com/Whiteknight/parrotsharp
[Winxed]: http://code.google.com/p/winxed

