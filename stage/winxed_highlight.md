---
layout: post
categories: [Parrot, Winxed]
title: Winxed Syntax Highlighters
---

For a long time I've been using JavaScript settings and syntax highlighters
in my editor to work with Winxed code. The overlap between languages is not
very large and is getting smaller with each new commit, so this solution was
clearly not going to be a long-term one.

Today I started writing
[syntax highlighters for gtkSourceView 2.0 for Winxed][winxed-highlight].
the highlighters are not too complicated, and are definitely missing a lot of
important things, but it's already a far cry better than what I had been
using. If you are using an editor based on gtkSourceView 2.0 (gedit, medit,
scribes, etc) and you are using Winxed, give the new syntax highlighters a
try.

[winxed-highlight]:

I may start putting together highlighters for PIR and maybe even NQP if I have
time (which I don't).
