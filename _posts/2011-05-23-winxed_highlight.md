---
layout: post
categories: [Parrot, Winxed]
title: Winxed Syntax Highlighters
---

I've been without internet at home for a few days, so I haven't been writing
up any new posts in a while. This is a short draft from a few weeks ago that
never got published so I'm putting it up now to pass some time. I *might* be
able to get back to regular posting and hacking tonight or tomorrow.

For a long time I've been using JavaScript settings and syntax highlighters
in my editor to work with Winxed code. The overlap between languages is not
very large and is getting smaller with each new commit, so this solution was
clearly not going to be a long-term one.

A few days ago I started writing up
[syntax highlighters for gtkSourceView 2.0 for Winxed][winxed-highlight].
the highlighters are not too complicated, and are definitely missing a lot of
important things, but it's already a far cry better than what I had been
using. If you are using an editor based on gtkSourceView 2.0 (gedit, medit,
scribes, etc) and you are using Winxed, give the new syntax highlighters a
try.

[winxed-highlight]: https://github.com/Whiteknight/winxed-highlight

I may start putting together highlighters for PIR and maybe even NQP if I have
time (which I don't). If anybody else has written up source code highlighters
for other Parrot-specific languages, please let me know about it. These would
be awesome things to start aggregating together. I know there are a few things
in the Parrot repo, but it's a very thin selection.
