---
layout: post
categories: [Parrot, Rosella, Template]
title: Compiling Templates
---

I had a precious few hours to myself yesterday and was able to do some updating
work on Rosella's Template library. I was able to use the time to implement a
feature I had wanted for a while: template compilation. You can now compile a
template into winxed code or even compile it all the way down to a Packfile.
Actually, that's sort of a lie. The Winxed compiler `.compile()` method returns
an Eval PMC not a PackfileView PMC. I'm going to submit a patch for that soon,
and when I do you'll be able to save the template to a .pbc file.

Here's how to use the Template library in the basic, interpreted way:

    var engine = new Rosella.Template.Engine();
    string output = engine.generate(template, context);

The template is a string with a format that I've demonstrated before, and the
`context` is any user-defined data structure that you want to use to populate
the variables in the template. I won't go into detail about those things in this
post. Now, after my recent changes and additions, you can compile your template
to an executable Sub:

    var engine = new Rosella.Template.Engine();
    var sub = engine.compile(template);
    string output = sub(context);

Or, if you really want to see the generated winxed code, you can get that:

    string wx_code = engine.compile_to_winxed(template);

The compilation process does take some time, there's no way to deny that. There
are ways to mitigate that expense, of course. You can compile ahead of time and
save the code to a file or even a .pbc and execute that later. There are several
strategies if you're really interested, I won't go into too much detail here.
Once the code is compiled, which can and should be done ahead of time, the time
savings during execution are significant. Here's some benchmarking I've done
to time a relatively simple template with a ten thousand iteration
`<$ repeat $>` loop:

    Interpreted:
    0.969569s - %100.000000
    0.796563s - %82.156419 (-%17.843581 compared to base)
    0.900937s - %92.921402 (-%7.078598 compared to base)

    Compiled:
    0.365500s - %37.697161 (-%62.302839 compared to base)
    0.498571s - %51.421914 (-%48.578086 compared to base)
    0.367616s - %37.915399 (-%62.084601 compared to base)

In some cases, pre-compiling the template can take a third as much time as
interpreting the template directly. Almost every timing I've seen is around 50%
or better of the interpreted time.

This is a very new feature and I haven't added it to the test suite yet. Expect
rough edges as I play with it and optimize it. If you're doing a lot of
templating with Rosella, this feature could help save some time for you and plus
it was a really fun thing to work on.
