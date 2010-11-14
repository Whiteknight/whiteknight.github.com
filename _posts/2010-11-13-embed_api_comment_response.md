---
layout: post
title: Comment Response, Embed API Design Review
categories: [Parrot, Embedding, ParrotProductManagement]
---

A few days ago I [posted a response][firstresponse] to a comment made by Parrot
hacker Peter Lobsinger. After I drafted that up, he posted a second comment
that deserved an even longer response from myself. Unfortunately, it's very
unweildy to write such long replies in the comments (even with my improved
Disqus comments), so I've turned my reply into yet another post.

[firstresponse]: /2010/11/11/problem_with_imcc.html
[secondcomment]: /2010/11/10/embed_api_sneak_preview.html#comment-96069446

> I very much dislike 2-stage interp initialization. From an embedder's point
> of view, it's a level of complexity I don't want to have to deal with. From
> parrot's point of view, it's a level of access we shouldn't be exposing.

I agree with you too. The first time I prototyped a 2-stage initialization
sequence I was unhappy with it. In the latest code, the 2-stage initialization
is out and I've replaced it with a structure-based initialization. This code
is committed and visible in my fork right now if you want to see what it
looks like.

It's not perfect, but as I show below, there aren't really too many good
alternatives.

> However, I do not like the struct/size based Parrot_api_make_interpreter
> approach much either (from Parrot). I would prefer a key-value list of some
> kind (length/array-of-pairs, length/va_list-of-pairs, json-string, alist,
> etc). Callers only allocate what they need, parrot can
> add/add-experimental/deprecate/remove features hassle-free.
>
> Also, a kv-pair interface could easily be carried over to allow
> fakecutables (which are yet another embedding case) to provide an argument
> processing preamble in an HLL (accepts argv, returns interpreter settings).
> Providing this would give fakecutables much more control over the main
> interpreter. HLLs could use this to provide command line arguments similar
> to those available only from parrot.exe today (or lesser, but still cool,
> things such as installing a static setting that is better suited to the
> language than the parrot default).

A C structure *is* a way to do a name/value mapping, so that's what I think
we should stick with, barring something obviously better.

In MATLAB, it's idiomatic to pass named parameters to a function using
variadic arguments like this:

    some_func("name1", value1, "name2", value2, "name3", value3, ...)

This is not a bad system, really, though I don't necessarily think we should
be borrowing syntax or semantics from MATLAB. One benefit that it has that
C doesn't, of course, is an ability to count the number of variadic arguments
passed to the function and loop over them. In C, we would need to pass a
count, or a format string, or something, and that starts to get ugly and
difficult to maintain quickly. When the embedder wants to specify a new option
they need to compute and pass the value, and also update the count. That's
a recipe for disaster and many bug reports.

At the lowest levels, a C struct really is a C array. The difference is that
the indices are calculated by the compiler at compile time instead of being
specified by an integer. I don't see any benefit whatsoever in using an array
or another similar structure to hold this information when a struct works
perfectly well. That is, this:

{% highlight c %}

opts[GC_SYSTEM_NAME] = "MS";
opts[GC_THRESHOLD] = 40;

{% endhighlight %}

is no better than this:

{% highlight c %}

opts->gc_system_name = "MS";
opts->gc_threshold = 40;

{% endhighlight %}

If anything, the second is better because the compiler can do type checking
for us. In the former example, everything would need to be a `void *` or
casted to it.

Also, we're talking about initialization options that occur before the interp
(and the GC) is set up, so we obviously can't use a Hash PMC, or anything
friendly like that.

Is it ugly? Yes, but I would argue that all uses of structs, and most C
syntax is ugly to some degree. I *am* open to other ideas here, but it seems
like we're going to be jumping back and forth between shades of lousy. This
is a case where I don't think we are going to find a perfect answer, and we
need to just be resolved to put *something* out the door eventually.

> You seem to eschew return values from C functions even when a function's
> only purpose is to compute a value (eg: Parrot_api_get_compiler()). I don't
> see why. It seems unnatural.

Almost every routine in Parrot actually has multiple possible results: It
can return a value you are looking for, it can return no value, it could fail
completely due to an unhandled exception, etc. You've mentioned this to me
recently, so let me repeat the words "semi-predicate problem" to you before I
say anything else.

For every API call, we need to indicate whether the call failed or succeeded.
In the case of success, we need to return whatever values are required (0+)
and in the case of failure, we need to indicate *why* it failed. This is a lot
of stuff for every API call.

When I say a call "failed", what I really mean to say is that somewhere along
the line Parrot threw an exception that was not handled. In Parrot master
we call the internal routine `die_from_exception`, which calls `Parrot_exit`,
which calls the C standard library function `exit`. This is obviously
unacceptable. In my fork when Parrot finds an unhandled exception, it
jumps back to the last API function call that was entered and signals failure
to the embedding application. The embedding application is then free to
request more information about the failure (`Parrot_api_get_last_error`, which
I will discuss more below), or try something else.

Of course, the embedding application could define it's own exception handler
of last resort and avoid the problem of Parrot throwing an unhandled exception
entirely. That's definitely the better way to do things, but I hardly think
it is something that we should mandate for all embedders. The alternative is
to define our own last-resort fallback mechanism and attempt to exit Parrot
as gracefully as possible.

What we could have for a return value is some kind of structure

{% highlight c %}

typedef struct {
    INTVAL status;
    INTVAL result_type;
    union {
        PMC *pmc_result;
        STRING *string_result;
        INTVAL int_result;
        FLOATVAL float_result;
    } value;
    STRING *errmsg;
}

{% endhighlight %}

and we could return this for every single function which could possibly fail
on an unhandled exception. However, this ignores functions like
`Parrot_ext_call(interp, sub, fmt, ...)`, which can legitimately return
multiple return values, and does so through pointers in the parameters list.
It seems like a weird break in consistency for every API call except
`Parrot_ext_call` to return one thing, and for that function to return
something different. We could be prepared to offer a single special case (I
doubt there will be just one function like this, but I'll simplify for the
sake of argument), but I don't like the break in consistency.

I don't eschew function return values. In the new API I mandate that every
function *must* have a return value and that all of these return values are
used for a consistent purpose: to communicate the success/failure status of
the call. In some places this is redundant or unnecessary, such as when the
embedding code defines it's own last-resort exception handler. If they don't,
we need to be prepared with a backup plan. Any Parrot operation which touches
a PMC might actually be working with a user-defined subtype and for any
VTABLE might be calling an override defined in PBC. That PBC might throw an
unhandled exception, which means for all but the most trivial API functions we
need to be prepared to handle that.

For the few API functions that don't require this kind of protection (yet)
I keep the same interface for consistency. We can argue whether that is
worthwhile or not, but it does help to insulate the user from knowledge of
our internals. We don't want them to know whether some routines are "trivial"
or not, and we don't want them relying on that if we need to make things more
complicated in the future. We've all got big things planned for Parrot in the
comming months and years, so it's foolhardy to think that anything which is
simple now won't become more complicated. We don't want to be changing
our interface every time our internals change. The point of an interface is to
encapsulate those kinds of details and hide them from the user.

Every function in the new API is defined following this pattern:

    INTVAL status = Parrot_api_do_something(<args>, <&results>);

I like the consistency and flexibility of this method, and the more I use it
the more natural it feels to me.

> I'm not a fan of Parrot_api_last_error_message() (gut feeling, from an
> embedder perspective), but I didn't know of a better interface.

I got the inspiration from the `GetLastError()` function in the Win32 API.
That is another example of something that we don't want to copy from too
heavily. But, like yourself, I can't really think of anything better. From
almost every API call we need to communicate whether it failed and, if so,
provide an error code or error message. If Parrot had i18n support, maybe
a non-zero status could double as a message ID slug that we could look up in
a table of message translations. But even that case fails because user code
could be defining their own messages and might not be properly inserted into
a translation table.

Again, I'll keep repeating this point (like a Parrot?): If the user defines
an exception handler to fall back on, none of this will ever matter. The
custom handler will receive the raw Exception PMC with all the detailed call
information and `Parrot_api_get_last_error` will be unnecessary. This function
is just part of our fallback strategy for when this hasn't happened.

We could change every API function to have this kind of
pattern:

    INTVAL status = Parrot_api_do_something(<args>, <&results>, &errmsg);

or

    void *first_result = Parrot_api_do(<args>, <&other_results>, &errmsg);

or any other permutation of these things. My goal is one of consistency, and
to a lesser degree, one of aesthetics. As I said, it is difficult to do pretty
things in C, but a man can dream.

> What
> follows is a survey of existing interpreter's embedding error handling
> interfaces. Perl has been omitted because I think we can all agree to do
> better.
>
> Lua[1][2][3]:
> longjmp() to inner-most pcall() or cpcall frame, falling back on panic()
>
> MzScheme[4]:
> longjmp() to inner-most scheme_setjmp() frame, I was unable to find any
> specification for fallback
> MzScheme appears to have a fairly nice/consistent/thought-out embed/extend
> interface that interacts with their GC and continuations. We should steal
> ideas.
>
> Ruby[5]:
> longjmp() to inner-most rb_protect(), falling back to exit()
>
> Python[6]:
> PyErr_Fetch(), PyErr_Clear(), and friends
>
> Another fairly good way to compare these language's embedding interfaces
> would be to look at an application that embeds them all to perform identical
> tasks (eg: vim, apache, etc).

doing a longjmp back to the inner-most embedding call is basically what I do
now, but I haven't added support for nesting. Fallback now is to call
`exit()`, but I may replace that with panic or something. Every API call
ensures that the stack-top pointer is defined too, for GC friendliness (though
maybe not as friendly as MzScheme). If the user defines their own stacktop
pointer, we will use that and do a full stack-trace of their code until that
point. If not, we define a stacktop pointer of our own in every API call in
so that our GC doesn't do something horrible.

There's a lot of work to be done internally to
Parrot before we can be more comfortable mixing API calls with continuations,
and I sincerely hope that Lorito either solves some of those problems or makes
the solutions easier to get to. We can't wait until that time before we put
together an embedding API, however.

One thing is certain, I do need to read over all those referenced docs before
I merge anything into Parrot master. I suspect that we aren't too far off from
what is being done elsewhere, though this interface is going to have alot of
maturing to do over time to match the capabilities, robustness, flexibility,
and elegance of some other solutions.

> From a parrot perspective, it makes more sense to specify a common compiler
> interface at the PIR-level. It shouldn't be that hard to provide a native
> compiler to satisfy this interface using NCI. In terms of calling in to a
> compiler from native, we could provide a convenience wrapper
> (Parrot_api_compile_file()), but if the interface is simply a function call,
> it is only really a nice-to-have.

I'm not really sure what you mean by "a common compiler interface at the
PIR-level". The short and snarky version of my reply is: "PIR is a dieing
animal". The longer version is that there should be some kind of standard
interface to a
compiler object. Almost anything you can do from PIR you can also do from C,
so it doesn't make a lot of sense to suggest that we should have a PIR
interface to compiler objects but not a C one. Also, for a simple embedding
application like the Parrot executable, do we need to have a small PIR thunk
to create and call the compiler before we create the compiler and become able
to compile the PIR in the first place? Maybe I'm not understanding what you
mean.

PCT-based compiler objects have a number of methods, including `.'compile'()`
which does most of the hard work. The system PIR compiler returns a Sub PMC
directly which has no methods of it's own and is invoked directly. We *do*
need a common interface for compilers at the object level. Once we have that
we can create API function calls which call VTABLEs and METHODs on that object
in a standard and reliable way.

[1]: http://www.lua.org/pil/24.3.1.html
[2]: http://pgl.yoyo.org/luai/i/lua_pcall
[3]: http://pgl.yoyo.org/luai/i/lua_cpcall
[4]: http://docs.racket-lang.org/inside/overview.html
[5]: http://aeditor.rubyforge.org/ruby_cplusplus/index.html
[6]: http://svn.python.org/projects/python/trunk/Include/pyerrors.h
