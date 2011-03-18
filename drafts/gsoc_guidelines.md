The news is in today: The Parrot Foundation is officially accepted as a
mentoring organization for Google Summer of Code 2011!

To help kick off GSoC in 2011, I'm going to publish a series of blog posts
about potential project ideas. I'll be posting ideas *rapidly*, maybe several
per day. I call this the shotgun approach: Put out a lot of ideas early to
help inspire students to become interested in certain areas early and then
put together better, more well-researched proposal.

The ideas that I post here on my blog do not need to be used by students
verbatim, and this also isn't a restrictive list. Students should feel free
to take as much or as little inspiration from this blog and other sources as
needed. feel would be suitable and desirable for a GSoC project on Parrot.

For a complete listing of project ideas from several members of the Parrot
community, check out the [list on the Parrot wiki][gsoc2011_tasklist].

[gsoc2011_tasklist]: http://trac.parrot.org/parrot/wiki/GSoc2011

At the end of last year's Google Summer of Code Program, Parrot ended up with
several branches of student work in various conditions which had not merged
into trunk. At the time of writing this, there are still 4 branches left over
from that program which have not yet been merged.

Over the past couple of months we've been slowly trying to change the way
Parrot does business. One of the most important and most visible changes is
the new way we are handling roadmap items. No longer does our roadmap
represent an almost fanciful list of things we would like to see happen. Our
roadmap now is a list of things which we are firmly dedicated to deliver
on or before the target deadline. Under promise, over deliver. And most
importantly, if we say we are going to have something done by a particular
time we *mean it*.

GSOC last year and the years before that as well were suffering from similar
problems to what our roadmap goals were seeing: Too much ambition, and too
little plan for actually delivering. In the end there are some projects which
were never completed (gsoc_threads), some that were completed but we didn't
want to merge (gsoc_nfg), and some that were completed but we didn't know
where to put them (gsoc_instrument and gsoc_past_optimization).

This year, we're doing something a little bit different. This year, we want to
try to pick more modest projects, and create a firm plan for when and how to
integrate them into Parrot. Every project, in order to be considered for
acceptance, must be able to be completed and integrated by the end of the
program (or shortly thereafter) with a very high degree of confidence. If you
can't tell me with something like 90% confidence that your project will not
only succeed, but succeed in the time alotted, you don't get the green light
for it.

In the coming days I'll be posting several ideas for new projects, including
estimated difficulty levels. These scores will be in the range of 1-5, with
1 being the easiest and 5 being the hardest, as the task is written. Parrot
will be very hesitant to accept a proposal at the 5/5 difficulty rating,
except in the case of extremely well-qualified students who have an existing
history of making high-level contributions to the Parrot project. For most
students, we will want to see proposals in the difficulty range of 1/5 through
3/5, including plans for how to fill the extra time if the main project is
completed early. The lower the difficulty level, the more "Plan B",
open-ended work you should account for in your proposal.

As I mentioned before, the ideas I post here on my blog are just suggestions
and inspirations. If you find a project which interests you, but has too
high an estimated difficulty level, you can do one of two things:

1. Propose a subset of the idea, or a smaller version of it. For instance, if
   the task idea is to "create a compiler for language X", you could propose
   to create a compiler for a subset of that language, or produce a compiler
   with no runtime library, or create a compiler by borrowing large amounts
   of existing open-sourced software (For instance, using [cafe][] as a
   pre-made basis for a JavaScript compiler).
2. Become a good salesperson and convince us that the project is not as
   difficult as we estimated, or that you have certain qualifications that
   make it more likely for you to complete the task than the average student.

[cafe]: https://github.com/zaach/cafe

Likewise, if you find a project with a low estimated difficulty, you can
expand on that idea to make a larger project at a higher difficulty, if you
think you can still get it done over the course of a summer.

Starting tonight I'm going to be posting potential project ideas as quickly as
I can possibly type them up. If *you* find a project that interests you, send
me a message, send a post to the parrot-dev mailing list, or hop onto the
Parrot IRC channel to start talking about it. The more you talk to us and
we talk to you now, the better we will all be when it comes time to submit
proposals.

Good luck!
