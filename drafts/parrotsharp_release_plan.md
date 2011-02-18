[ParrotSharp][] is a [project I started][parrot_sharp_start_post] during the
[embedding API project][embed_api_project]. It's a library of bindings for
embedding Parrot into C# programs, and being able to call into Parrot from the
.Net CLR.

[ParrotSharp]: https://github.com/Whiteknight/parrotsharp
[parrot_sharp_start_post]: /2010/12/14/introducing_parrot_sharp.html
[embed_api_project]: /2010/11/05/embedding_api_team.html

ParrotSharp has been coming along quite nicely, and after GCI it now has an
[NUnit-based test suite][nunit] which I've been slowly expanding to cover some
of the more interesting bits of functionality.

[nunit]: http://www.mono-project.com/NUnit

What ParrotSharp does is basically two things. First, it provides the
low-level native call bindings to access the Parrot embedding API routines
from C#. These raw interfaces are a little bit messy, since most of the data
types are passed around as `System.IntPtr` types. ParrotSharp also provides a
number of wrapper classes which take these opaque pointers and use them to
expose some basic functionality.

At the moment, I've only provided wrappers for a scant handful of Parrot
types, mostly the primitive ones I've needed to set up the most basic
operations so far: Null, Interpreter, Sub, String, Integer, and Exception.
Eventually I'll be providing a number of other types as well, although I will
probably not provide wrappers for every single Parrot type. In some cases such
as Sub, it doesn't necessarily make sense to provide a separate Coroutine type
as well, since the two are almost completely identical in terms of interface.
Also, I will probably only provide a single type as a wrapper for all the
Parrot array types. Since the embedding API wraps most keyed indexing operands
up as PMCs, it doesn't make any difference from an interface perspective
whether we're working with a ResizablePMCArray or a FixedStringArray.

Part of me was hoping I would be able to put together some kind of a release
of ParrotSharp soon, to be able to target the 3.0 stable release of Parrot.
However, I've now decided that this won't happen. ParrotSharp really doesn't
expose nearly enough functionality to be usable for more than small toy
programs yet. It doesn't provide access to most of the Parrot embedding API
routines, and it doesn't provide access to most necessary features such as
custom object types. I'm working on all these things, but it's going to take
a while before I'm comfortable calling what I have a "useful release".

I would really like to be able to provide a usable replacement for Parrot's
current frontend executable written in C#. However, because of problems with
the way IMCC is currently used in the embedding API this really isn't possible
to do in a clean way. This is really going to rely on some of my IMCC cleanups
and improvements being merged in first, before I can start doing PIR
compilations from my C# programs. Here is some example code that shows what
ParrotSharp can do right now:

{% hightlight c# %}

class App {
    public static int Main(string[] args) {
        Parrot interp = new Parrot();
        IParrot_PMC pbc = interp.LoadBytecodeFile("foo/bar/baz.pbc");
        interp.RunBytecode(pbc, interp.PmcNull);
        IPMCFactory factory
            = new PMCFactory<MyCustomPMCType>(interp, "MyCustomPMCType");
        IParrot_PMC baz = factory.Instance();
        baz.InvokeMethod("FooBar");
        IParrot_PMC sig = CallContext.GetFactory(interp).Instance();
        sig.StringValue = "PiPSP->";
        sig[0] = baz;
        sig[1] = interp.PmcNull;
        sig[2] = "Test";
        sig[3] = 127.ToParrotIntegerPMC(interp);
        IParrot_PMC meth = baz.FindMethod("BarBaz");
        meth.Invoke(sig);
    }
}

class MyCustomPMCType : Parrot_PMC
{
    public MyCustomPMCType(Parrot interp, IntPtr ptr) : base(interp, ptr);
}

{% endhighlight %}

I can create a variety of PMCs, I can create signatures, call functions and
methods with parameters, and do a bunch of automatic conversions between
Parrot and C# types. I cannot currently return values from called Subs, do any
kind of arity checking on signatures, or do anything too fancy. I do have
keyed access for array and hash types, but I don't have access to
VTABLE_elements or VTABLE_get_iter yet to do anything useful with them.

ParrotSharp also only works with Mono right now, I haven't made it work with
VisualStudio quite yet. I think that's something I am going to have to do
in time for any kind of "real" release.



