---
layout: post
categories: [TextEditors]
title: Text Editors, Edition 2
---

Every couple of months I get unhappy with my text editor situation again, and
go out searching for a new one. Currently I use [medit][], although I have a
number of complaints about that software. I don't use medit because I think
it's a great editor that meets all my needs. I use medit because it appears to
be marginally better than [gedit][] and doesn't have some of the glaring flaws
that other editors I've tried use.

[medit]: http://mooedit.sourceforge.net/
[gedit]: http://projects.gnome.org/gedit/

A long time ago I put together a list of requirements I have for a text editor.
I've been able to narrow that list down quite considerably. Here's a list of
things which I require, and I won't use an editor without:

1. Must be able to use the [solarized][] color scheme without heroic effort on
   my part. Solarized is a popular and decent theme. It's not perfect for me,
   but it is good enough and is becoming pretty pervasive. I like the
   consistency aspect of being able to use the same color scheme at work on
   Windows and at home on Linux. Basically, this is a requirement that the theme
   used is both configurable, and can be saved to a file and transfered between
   computers.
2. The editor must be light-weight. I'm not interested in using a huge IDE for
   most of my work.
3. Line numbering.
4. Multiple documents in a single window. Tabbed is good. Other schemes might be
   fine too. There should also be easy key-bindings for moving between open
   documents.
5. Must be able to replace tabs with spaces. Specifically, I use 4 spaces, so
   that must either be the hard-coded default or must be a configurable option.
6. Must work well with the languages I use regularly (C, C#, Winxed). If it
   can't offer perfect support, which I wouldn't expect for fabricated
   languages, it should at least fail gracefully. Preferably, the editor should
   provide the ability to touch-up problems with new languages, though that's
   not a feature I've ever seen outside of Notepad++ on Windows.
7. Common keybindings, or easy-to-find-and-use plugins to add proper
   keybindings. I don't want to waste 30 minutes of my life researching some
   custom script language, then writing and debugging code to convince my
   editor that ctrl+S is, and always should be, a command to save the document.

[solarized]: https://github.com/altercation/solarized

I don't feel like this is too huge a list of requirements, and most editors
either do, or can through basic configuration, provide all these things.

I've been using medit. It's a nice enough editor, although it's hard to get
excited about it. I was under the impression that it was at least marginally
better than gedit, but on second thought it might not be. gedit does have some
things that I seem to quite enjoy: spell check is quite a nice feature when I
am writing long rambling blog posts for instance. I always end up inventing
enough new words, purposefully misspelling existing words, using jargon, and
using all sorts of weird proper names (like IRC handles) to a degree that makes
spell check less useful than you would expect. I also rarely see spellcheck
systems which can be used with code in a non-trivial way. I *have* seen such
systems, but they are certainly not common enough.

medit has better key bindings than gedit does and, more importantly, has an easy
system to edit key bindings. With a configuration system, I can add the things I
want, even if the defaults are not perfect. gedit doesn't allow me to edit key
bindings, and some of the ones it does use (Ctrl+Alt+PgUp to move to the next
tab, for instance) are really sub-optimal. medit also allows me to assign file
extensions to existing syntax highlighting schemes, which is very important if
I am editing an NQP, Winxed, or PIR text file. gedit doesn't seem to have any
automatic way to associate those files with existing syntax highlighting
schemes, which is a big pain to have to do manually for every file I open.

Like I said, medit fits my needs marginally better than gedit does, although
there is no clear-cut winner between them in all categories. This blog post
was actually written in gedit, but most of them are written in medit.

One day, when Parrot has proper bindings to GObject, I would love to write a
Parrot plugin for gedit. Once I have that ability, and am able to write my own
plugins in any parrot language of my choice (Winxed anybody?) I'll be much
happier with it.

Every few months I give [vim][] a try. I install it on my system, open it, use
it until my blood pressure rises to medical emergency levels, then uninstall it
again. I don't use vim because of the key bindings, period. I can't do what I
want to do in vim, and I am not going to spend hours trying to hack things which
I consider "normal" and "standard" into my `.vimrc` file.

[vim]: http://www.vim.org/

Recently I've found [cream][], which is a configuration system for vim which
adds in all the standard features that one would expect from an editor. Cream
is a very interesting editor, and I used it for most of a day. Far longer than I
have ever been able to put up with basic vim. The problems I have with cream are
that I was not able to configure it to insert spaces instead of tabs, and the
fact that it uses a custom theme system which does not appear to be compatible
with normal vim color schemes. The cream FAQ does say how to set it up to use
spaces instead of tabs, but that sequence did not work for me on two separate
machines. If I could get tabs working the way I expect, could get solarized
working on it seamlessly, and maybe figure out a few things, I might consider
moving to cream full time. It does work on Windows, and that portability is very
attractive to me.

[cream]: http://cream.sourceforge.net/

I'm not a praying man, but if I were I would pray to never have to use emacs
again. I've heard tell that emacs is like an entire operating system, though one
which lacks a good text editor. I've never found reason to disagree with that
sentiment.

I've been playing with [scribes][], which is a very promising-looking editor.
It is obviously still early in the development cycle, but it has got a lot of
potential and I was quite impressed by it when I used it. It does use the
`gtksourceview` control which medit an gedit both appear to use, which means it
can easily use any themes I've installed for gedit to use. Configuration options
are very light, but I have been able to set it up mostly the way I like. What I
don't like about scribes is that it doesn't have tab-based editing, and there
are a few bugs I found during my tests. If it can add a few more necessary
features, squelch a few bugs, and mature a little bit, I would be very happy to
use this on a more regular basis.

[scribes]: http://scribes.sourceforge.net/

Here are a few other text editors I've played with recently:

* **jEdit**: Obviously written in Java, because the whole UI has that ugly
  java look to it. Plus, I think it's only designed to edit java code, which I
  do not use.
* **MonoDevelop**: I use MonoDevelop when I am writing C# on Linux. It's enough
  like VisualStudio that I feel comfortable with using it, and it has all the
  features you would like to see in a C# IDE. I haven't really explored putting
  solarized on it, but I also didn't see any obvious easy way to do it. I don't
  use it for any other purpose besides C#, so most of my requirements for a
  general-purpose editor don't apply.
* **SciTE**: This editor both looks extremely ugly and has no obvious, easy way
  to modify syntax highlighting away from the default theme (which is *not* the
  way I like it). I haven't spent much time using it because of those two
  reasons alone.
* **Kate**: I work on Gnome mostly, although I have been contemplating a move to
  Kubuntu. KDE 4.6 looks quite compelling, but then again so do Unity and Gnome
  3 (once those two things get a few kinks worked out). I find that Kate doesn't
  fit into my work flow very well. It's difficult to use from the console
  because it spits out a lot of debug/error text for some reason. It has
  configurable syntax highlighting, but has no easy way that I can find to save
  settings to a file, and load settings in from a file. Maybe such a feature
  does exist, but the documentation for it is unnecessarily hidden or obscured.
  Kate on my machine has a weird bug where the cursor is the same color as the
  editor background, which means I can't see where the cursor is. I don't know
  if that's a problem with Kate, the underlying Qt libraries, or the attempt to
  use Kate from Gnome.
* **TEA Text Editor**: Like SciTE, this one is extremely ugly looking. On top of
  that, it hard codes some colors like the menu option color to black. Using TEA
  on Ubuntu with the modern purple-and-black color scheme means that most menu
  options are invisible, unless I go into the options screen and set the UI
  theme to something less ignorant. This editor does seem to have a lot of
  configuration options, but doesn't appear to have any real system for plugins
  nor any easy way to save and import themes. It does seem to have a few
  interesting features and ideas, but none of them really matter to me.
* **PyRoom** I like the idea of this editor. It's minimalist and
  straight-forward. It's just as minimalist if not more so than scribes, but
  you lose out on some of the features and usability of scribes. It's a decent
  idea if you just want to write text and don't want distractions, but I feel
  like this takes that idea to an unwelcome extreme.
* **Padre**: I love the idea of padre, but it really isn't what I want. It looks
  great, but I can't find an easy way to use custom syntax highlighting themes.
  It does have support for other languages, but this is a Perl IDE for writing
  Perl code by Perl coders. It does have tons of options, lots of
  configurability, and a growing list of plugins available on CPAN, so maybe
  one day I'll come back to it.

So that's my quick survey of the field. For the forseeable future I will
probably stick with medit and gedit. However, cream and scribes are squarely
on my radar and are attractive alternate options. If any of those things gain
a Parrot plugin for writing utilities in Parrot languages that might make my
decision for me. Unforunately, I suspect that if any of them do get such a
plugin I'll probably be the one writing it.
