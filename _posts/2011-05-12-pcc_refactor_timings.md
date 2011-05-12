---
layout: post
categories: [Parrot, PCC]
title: PCC Refactor Timings
---

To follow up with two of my [previous][pcc_refactors] [posts][vtable_timings],
I wanted to do a little bit of benchmarking to see what performance savings we
could have (if any) if we started pursuing my idea for exposing calls directly
to the user and not doing the heavy-weight processing we do in `fill_params`
for every call. To do this benchmarking, I need to use PASM for some tests,
because PIR hides a lot of complexity and forces the use of `set_args` and
`get_params` in subroutine calls and definitions (and their analogs for
returns). Here is a PASM wrapper function that I am using for timings. The
code to benchmark, which I will show in shorter snippets in the rest of the
post will be copy+pasted in the "Inner Loop" portion:

    .pcc_sub :main main:
        get_global P1, "foo"
        set I1, 1000000
        print "Starting ... "
        time N0
      loop_top:


        # Inner loop
        # Code to time goes here


        dec I1
        if I1, loop_top
        time N1
        sub N2, N1, N0
        print "Total time: "
        print N2
        say "s."
        exit 0

    .pcc_sub foo:
        # Empty Sub. Return immediately and do nothing
        returncc

Likewise, here's the PIR equivalent, to show the differences between the two.
I've deliberately kept the PIR version almost identical to the PASM version,
so it's easier to compare directly:

    .sub main :main
        get_global $P1, "foo"
        $I1 = 1000000
        print "Starting ... "
        $N0 = time
      loop_top:


        # Inner loop.


        dec $I1
        if $I1, loop_top
        $N1 = time
        $N2 = $N1 - $N0
        print "Total time: "
        print $N2
        say "s."
    .end

    .sub foo
    .end

All the changes I talk about making to Parrot in this post are being performed
in the `whiteknight/pcc_prototyping` branch for now. I may include some of
these benchmarks there as well, although I do not do that currently. If you
want to follow along with this work, or you want to get involved with it, you
can do it in that branch.

For the first test on this machine, I'm using a direct `invokecc` call for
PASM vs a normal sugared call for PIR:

PASM:
    invoke P1

PIR:
    $P1()

And the timing numbers for just this call:

    Starting ... Total time: 1.40410900115967s.
    Starting ... Total time: 1.67736411094666s.

I've run this a few times, and while there is some variability (I'm running
this on a VM right now), the PIR version appears to consistantly take about
120% of the time to run that the PASM version takes. This is without any args
or any returns of any kind. Baseline is about 20% slower.

For the next test, I created a new op, a variant of `invokecc` which takes two
arguments: the Sub PMC to invoke and a CallContext which contains the
function arguments. Now let's compare:

PASM:
    new P2, "CallContext"
    set P2[0], 1
    set P2[1], 2
    invokecc P1, P2

PIR:
    $P1(1, 2)

Now the timings swing back the other way, with the PASM version being more
expensive to use:

    Starting ... Total time: 2.33223795890808s.
    Starting ... Total time: 2.07209300994873s.

Why is this? A big part of the problem is the use of the `new_p_sc` opcode,
which uses a string literal to lookup the type for CallContext. This is far
slower than it needs to be. Next, I added a new opcode called
`new_call_context`, which does a numerical lookup on the built-in type, the
same way that PCC would build it internally. Again, the numbers swing back the
other way.

PASM:
    new_call_context P2
    set P2[0], 1
    set P2[1], 2
    invokecc P1, P2

And the results:
    Starting ... Total time: 1.90112495422363s.
    Starting ... Total time: 2.06399393081665s.

Not as big a change as last time, but the PASM approach without `set_args`
or `get_params` still seems to be doing just as well or better if we add in
the correct infrastructure for it. One other important thing to demonstrate
is that when we compile these two files down to bytecode, the file sizes are
different. The PASM version is 1824 bytes, while the PIR version is 1888
bytes. In the disassembly, the two versions have the same number of opcodes
(16 total), while the PIR version has two additional PMC constants for
signature arrays that the PASM version does not have. When you think that
every single callsite in your program is likely to have an additional
signature array constant PMC, you can imagine how the costs of that mechanism
grows for non-trivial programs.

Let's try named parameters now:

PASM:
    new_call_context P2
    set P2["Foo"], 1
    set P2["Bar"], 2
    invokecc P1, P2

PIR:
    $P1(1 :named("Foo"), 2 :named("Bar"))

Results:
    Starting ... Total time: 2.52663016319275s.
    Starting ... Total time: 2.98304104804993s.

The PASM version still wins, although the margin is still not by much. Part of
the reason for that is because we aren't doing any argument unpacking in the
callee, which is where `fill_params` would be called, and where I think the
real costs are hidden. I'm going to expand out the `foo` subroutine to read
two positional arguments, as integers, and add them together. To get the raw
CallContext in the callee, I need to add a new `get_call_context` op to read
it directly without going through `get_params`:

PASM:
    .pcc_sub foo:
        get_call_context P0
        set I0, P0[0]
        set I1, P0[1]
        add I2, I0, I1
        returncc

PIR:
    .sub foo
        .param int a
        .param int b
        $I2 = a + b
    .end

Results:
    Starting ... Total time: 2.54172682762146s.
    Starting ... Total time: 2.62854504585266s.

No huge change here. The PASM version is still faster, still by a slim margin.
Let's repeat this with named arguments instead.

PASM:
    .pcc_sub foo:
        get_call_context P0
        set I0, P0["Foo"]
        set I1, P0["Bar"]
        add I2, I0, I1
        returncc
PIR:
    .sub foo
        .param int a :named("Foo")
        .param int b :named("Bar")
        $I2 = a + b
    .end

Results:
    Starting ... Total time: 3.31844401359558s.
    Starting ... Total time: 4.35232996940613s.

Ooopsies. Here PIR demonstrates the huge fail of `fill_params` processing
named arguments. PASM wins this round by a convincing margin. Almost a whole
second more time to run this test using the "old" style of dispatch over my
new version of it, when named arguments are involved. Again, multiply that
across all the millions of function calls in a non-trivial program using these
features, and you start to see the problem.

Let's try with optional/positional args:

PASM:
    .sub foo
        .param int a
        .param int b
        .param int c :optional
        .param int has_c :opt_flag
        $I2 = a + b
    .end

PIR:
    .sub foo
        .param int a
        .param int b
        .param int c :optional
        .param int has_c :opt_flag
        $I2 = a + b
    .end

Results:
    Starting ... Total time: 2.70708703994751s.
    Starting ... Total time: 2.70915794372559s.

The two are neck and neck. It seems that the `exists` opcode is more expensive
than whatever check `fill_params` uses in this case. Now, let's try making
those optional args named:

PASM:
    .pcc_sub foo:
        get_call_context P0
        set I0, P0["Foo"]
        set I1, P0["Bar"]
        set I2, P0["Baz"]
        exists I3, P0["Baz"]
        add I2, I0, I1
        returncc

PIR:
    .sub foo
        .param int a :named("Foo")
        .param int b :named("Bar")
        .param int c :named("Baz") :optional
        .param int has_c :opt_flag
        $I2 = a + b
    .end

Results:
    Starting ... Total time: 3.70791792869568s.
    Starting ... Total time: 4.56210494041443s.

... And here comes the fail train again. `exists` causes the PASM version to
become more expensive, but the costs of named argument processing in
`fill_params` for the PIR version is higher too.

I'm not going to grind this point into the ground any further, and I'm also
not going to analyze these numbers too hard. I ran each test a few times to
make sure the numbers were looking consistant, but I'm not going to compute
all the statistics. At the very least, what I can say is that my approach
of avoiding the `set_args` and `get_params` opcodes, and the `fill_params`
function is no worse and in some cases noticably better than the "standard"
PIR approach. I added three new opcodes, but should be able to delete four
of them in return, so Parrot is not getting any more complex because of it.

bacek pointed out yesterday that we probably need to keep `fill_params` around
as the mechanism for passing arguments between C and PIR. That's fine by me,
although I feel like we could be passing the CallContext to embedding users
as well and unpacking that manually using direct VTABLE calls in the same
exact way as I show above. That would enable us to kill `fill_params`
completely, and prevent the need for us to be passing around signature strings
at the C level to describe calls. That, unlike most of what I talk about
above, would require a deprecation cycle to implement.

Even if the timing numbers were all dead even, this approach still gives more
power and flexibility to the users of Parrot and allows Parrot to make fewer
decisions by default. Parrot doesn't need to add a mechanism for processing
arguments, because users can do it themselves. Compilers and code generators
like PCT and Winxed can be making decisions about calling conventions and
argument processing, and spare Parrot the headache of having to guess what
users need (and, in many cases, get those guesses wrong). It's worth pointing
out that Rakudo *already processes their arguments like this*, because the
Parrot system of `fill_params` and its semantics are not what Rakudo wants or
needs. When our biggest user tells us that our core calling convention and
argument processing mechanisms are not adequate, maybe it's time we open the
gates and allow users to do what they need to do themselves.
