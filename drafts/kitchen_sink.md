What really bothers me is the hyberbolic arccotangent. It's not that
specifically that I find bothersome. It's when I see it implemented in
runtime libraries and programming languages. People say to themselves "Hey,
Implementing basic trig functions like sine and cosine are easy, let's throw
them into our runtime library!". Where you have sine and cosine, of course
you need tangent as well. And if you have those three, it makes good sense
to include the inversese: arcsine, arcosine, and arctangent. And from there
it's a tempting and slippery slope to start adding more. Each one is so simple
by itself, and so useful in certain narrow applications, and we already have
a namespace for trigonometic functions, so we can just keep going!

It's not long before you're swamped in these routines. sine, cosine, tangent,
arcsine, arccosine and arctangent are just the beginning. Before long you have
the entire family of basic trig routines: secant, cosecant, arcsecant and
arccosecant. And then you add in the hyperbolics: hyperbolic sine, hyperbolic
cosine, hyperbolic tangent, hyperbolic cotangent, hyperbolic arcsine,
hyperbolic arccosine, hyperbolic arctangent, hyperbolic secant, hyperbolic
cosecant, hyperbolic arcsecant and hyperbolic arccosecant. Then, since you've
come this far, maybe we just go all the way. Add in versine, coversine,
haversine, havercosine, cohaversine, cohavercosine, exsecant and excosecant.
For good measure, throw in the inverses too, including the ones that we've
just invented for this purpose: arcversine, archaversine, arccoversine,
archavercosine, arccohaversine, arccohavercosine, arcexsecant, and
arcexcosecant. After all, each one of these is individually trivially easy to
write, and might be useful to somebody somewhere, one day, maybe. Once you get
started, just keep writing them, it doesn't hurt nobody, and it's your own
time that's being wasted, not mine.

Now you have all these dozens of trigonometic routines and, since you're a
good and diligent little coder, you have docs and tests for all of them too.
This includes tests for in-range and out-of-range, and boundry values as
well. Corner cases. What happens when you pass an infinity value to
arccohaversine? What happens when you pass NaN to hyperbolic arccotangent?
Are you testing whether the input and output ranges for each are inclusive or
exclusive? How do you handle rounding errors for values which may be floating
around the boundary? Are you verifying that output values are in the correct
ranges, and how are you performing that verification? After all, we all know
that equality in floating point numbers is tricky to test.

If the language you are using is high-level enough, or if it is dynamic, are
you accounting for various input types? Integers? Single- and Double-precision
floating point numbers? Fixed-decimal point numbers? Does your runtime
support complex numbers? Do you have a fractional type that isn't reduced
to floats or decimal numbers? All of these types of inputs are going to need
their own set of tests, remember. Oh, and don't forget all the documentation.
Users care strongly about having good documentation.

Trigonometric routines are just one example of "kitchen sink" mentality. It's
like those people on TV who hoard, and fill their houses up to the brim with
useless garbage. It starts off easy enough: a collection of bottle caps and
a pile of snail mail that you don't want to throw away without looking at, but
you never seem to have the time to look at it. And then you plan a yard sale,
but it rains that day and you don't reschedule. Before you know it, your
house is filled with crap, you can't find your furniture, and you have to eat
dinner standing up in the the path between where the kitchen used to be, and
the room where a TV is buried somewhere.

The thinking goes that it doesn't hurt anybody. Each of these little routines
is trivially easy to write, and they don't take up much space, and some
people might appreciate it, and there's no real reason not to. Each one is so
small and easy and benign!

The problem manifests itself when you have hundreds of such routines spanning
several domains and namespaces. The list of trigonometic routines is limited,
but the field of mathematics can easily supply more. Throw together a quick
routine to calculate greatest common denominators between fractions, and a
routine to calculate least common multiples for factorizations. Then a routine
to factor a number into primes, and a routine to calculate logarithms for
various bases, one to calculate the tower function, fibbonaci and leonardo
series, bessel functions and gamma functions and a routine to approximate
the number of primes below a certain threshold, and a routine to convert
uniformly-distributed random numbers into normally distributed random numbers,
and the list goes on.

Any given routine may be trivial to implement, but that doesn't mean you need
to include it in your software. Any given routine may have near-zero cost
in terms of development, maintenance and runtime performance, but many
hundreds and even thousands of routines with near-zero cost add up to
significant problems. Every time somebody downloads your software, some of
that bandwidth is eaten up by your havercosine implementation, its
documentation and its tests. Every time somebody runs your program, your
GCD routine needs to be loaded from disk into memory. Many times when your
program goes to sleep, your implementation of the gamma function needs to
be cached to disk and then reloaded again if it's in a memory page containing
something that is used more commonly. Every time your prime number
factorization routine is in memory, some other bit of useful data isn't, and
the virtual memory manager is going to have to work just a teensy-weensy bit
harder.

I'm not saying we shouldn't have a myriad of routines, especially mathematics
routines, floating around to make life easier. I'm saying that you don't need
to include them all in your runtime by default. You shouldn't feel compelled
to include things because you can and because it doesn't seem like the costs
are high. Any costs aggregate when you consider millions of users running your
software hundreds or thousands of times. Instead, you should feel compelled
to avoid unnecessary code and the unnecessary costs that come with it. Take
all your dozens of trigonometic routines and push them out into a separate
library, and let the users who need that stuff to incur the costs of it. Users
who don't, and I suspect the vast majority of programs don't need to make any
use of all but the most common handful of trig functions, or other advanced
mathematical routines, should not have to pay for it.

When putting together a runtime, ask yourself for each and every item "is the
average user going to need or want this?" If so, it's a candidate for
inclusion. If not, find some other place to put it.

