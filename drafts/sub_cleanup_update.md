---
layout: post
categories: [Parrot, Rosella, Net]
title: Sub Flags Cleanup Work Update
---

The remove_sub_flags branch I've been working on with Parrot does one very
important thing: It changes all instances of the old `load_bytecode_s` opcode
to instances of `load_bytecode_p_s`, with some additional supporting logic.
As you might expect, this work has me touching lots of PIR files, especially
the various files in the runtime library. One thing I've noticed is that many
of these library files are either in bad condition or are not used anywhere
except in the test suite.

So I sent an email to parrot-dev suggesting we review the contents of our
runtime library and start removing cruft. Basically, anything that isn't
relied upon by HLLs and other projects, and anything which doesn't provide
any benefit over alternatives should probably go. At the least I would like
to see some of these libraries moved to external repos. Any that aren't worth
that effort can probably be deleted.

Rosella does already provide a large amount of functionality suitable to
replace many of the bits found in the standard library. In the future I would
like to expand it to replace even more. That's neither here nor there, of
course. Rosella isn't snapshotted into the Parrot repository so it's libraries
are not as readily available as the ones in the standard runtime.
