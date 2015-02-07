---
layout: post
categories: [Parrot, Embedding]
title: Embedding API Ready For Review
---

I've pushed a series of commits last night which complete the first round of
the Embedding API rewrite. These commits fix up error handling so that now the
embedding application is in charge of handling error messages and error
information instead of libparrot. libparrot no longer uses direct `fprintf`
calls to display error messages to the user. Instead, that information is
passed via an Exception PMC, to the embedding application for dissection and
handling.

At this point the embedding API provides the minimum functionality that I
think most simple embedding applications will need. It isn't perfect, in many
places it exposes far too much of the internal architecture of Parrot and
some of those details are not pretty. Parrot bytecode handling, for example,
leaves a hell of a lot to be desired. The API functions dealing with bytecode
are probably the most likely to change in the near future because of this.

A while ago I [talked about error handling][errorhandling], and saying that
most error handling could be simplified if the embedding application
registered a fallback handler. This would prevent any exceptions from being
unhandled and leading to the fallback logic in `die_from_exception` and
`Parrot_x_jump_out` and friends. Unfortunately in the API as it currently
exists, this isn't possible. Part of the reason is because of a shortcoming
in the exception system where I can't simply register a C function as a
handler. Instead we have to register a `parrot_runloop_t` structure as a
handler, which contains a reference to a `jmp_buf` structure. I don't know if
we will want the user to be passing in `jmp_buf` handlers through the API or
not, but I strongly suspect that the answer is "no". `jmp_buf` structures are
way too low-level for this kind of work, and the less we have to expose the
users to that, the better.

[errorhandling]: /2010/12/01/embedding_error_io.html

The branch is passing all tests, even the tests of questionable value that
use Perl5 regexes to match the exact text of error messages. In fact, one
old TODO test began passing and was moved off the TODO list permanently.

The new system duplicates the command-line user experience exactly, which
means we can avoid a deprecation boundary. This makes me very happy, and
opens the possibility that we could get it merged in before 3.0. We have a lot
of work to do before we can talk about any merger, but if we could get it in
before 3.0 that would be the best in my opinion.

When I say the branch is "passing all tests", what I really mean to say is
that it passes all *functionality* tests. Every test where code is executed is
behaving the same now as it is in master. Where we are still seeing failures
are in the coding standards test. In my haste to prototype and implement
various new ideas I completely disregarded almost every coding standard that
we have.

Almost none of the new work has proper documentation. I listed this as a
task for GCI, although no student has jumped at the idea yet. I'm hoping that
I can gind a student to do it so I don't have to. That would make me *very*
happy indeed.

Things like line-length limits are completely violated. With all the shuffling
of parameters, signatures, and even entire functions, I didn't want to spend
time worrying about line lengths. I've started going back to fix
this--slowly--but there is more work to be done.

I'm not consistently using the various assertion macros that most Parrot
functions are decorated with either. Macros like `ARGMOD()` and
`ASSERT_ARGS()` do play a valuable role in preventing developer mistakes and
in giving the compiler valuable optimization hints, but they can be a bit
of a pain to keep current when development is happening quickly. These things
do need to be added before we can merge.

Finally, though this doesn't cause a failed coding standards test failure
*per se*, it is test-related: We don't have *any* tests for the new API
functions in our test suite. We absolutely need to add several before we can
talk about merging. If I am smart I'll try to set up some GCI tests for this.
Before I can do that though, I will need to set up a few tests of my own first
to serve as a template the students can follow.

So where do we go from here? First we need to make the fixes I described above
and add plenty of test coverage. Also, I've gotten a report that the branch is
having some troubles on darwin/ppc, so I need to get those resolved too.
Obviously we need wide-spread community review of the work to make sure it's
actually something that we want to integrate and support. We may want to get
some significant community support built up before we spend the effort to
write a bajillion new tests, unless we want to live dangerously.

Once the branch finally does get merged, we need to make sure embedding
projects like PL/Parrot, mod_parrot, and other things work well with it. In
updating these projects I'm sure we will find new bugs, and identify holes in
the API where new functions need to be added. Adding a new API wrapper around
an existing function is trivial, though in many cases we may want to make the
API call a little smarter than just a dumb wrapper around an existing
function. Doing input validation is a small start. Being able to dispatch to
families of related functions depending on the inputs would be even better.
