---
layout: post
categories: [Parrot, Embedding]
title: Where do PMCs Come From?
---

"Mommy, where do PMCs come from?"

"Oh, well... You see, when a man and a woman love each other very very much...
Actually, go ask your father. He knows."

Where do we get PMCs from? This isn't really a rhetorical question. In
libparrot we can call the function `Parrot_pmc_new` with a type number and get
a PMC of that type. That's a terrible interface, and relies on the internal
knowledge that we store PMC type definitions in a numbered array. This is not
the kind of interface that we want to be using internally, in the long run. It
is certainly not something that we ever want to expose to users.

We also don't really want to be looking up types by name with a String,
because type names can be nested, and an interface that looks up a type by
String name is only useful for a fraction of types. I don't want any API
functions which aren't useful in a general case, and if I have a function that
properly handles the general case there is absolutely no reason to have one
that only handles a subset. So, no string-based name lookups. We want to look
up types using an aggregate, such as an array of strings.

To get a PMC, typically you first need to get the Class object for that type,
and then instantiate that. We would typically look up the Class using a Key or
String-array PMC which contains a list of nested names.

We use an array PMC to get the class PMC to instantiate a PMC. Easy-peasy.

"But wait!" you exclaim. "This creates a conundrum. A paradox even. How do we
get the array PMC of names that we use to look up the class PMC in the first
place?"

That's easy too: We look up the ResizableStringArray Class PMC, and
instantiate it.

I'm obviously being a little facetious here. Obviously we have a boostrapping
problem. We will never be able to create any PMCs if we must have a PMC before
we can create one. Somewhere along the line we need the ability to create a
PMC of a standard type so that we can get this process started. I could have
(and probably will eventually have) a function `Parrot_api_pmc_box_string`,
which would take a `Parrot_String` and return a String PMC. That's a pretty
standard operation, and one the API will likely want to support at some point.
However, like I mentioned above, we don't want to be looking up types with
simple Strings, because that isn't sufficently useful enough. We want to be
using arrays.

We do have a function already that takes an array of C strings and returns a
Parrot_PMC array type. I use this for passing the argv array from C to the
main function in bytecode. This function is named
`Parrot_api_build_argv_array`. In reality, I should rename it to
`Parrot_api_pmc_string_array` or something similar, and use it as a general
purpose routine for converting a C string array into an array Parrot_PMC. With
this utility, we are now able to create a PMC:

1. Create a C string array
2. Turn that into a Parrot_PMC array using `Parrot_api_pmc_string_array`
3. Lookup the Class with `Parrot_api_pmc_lookup_class`
4. Instantiate the PMC with `Parrot_api_pmc_new_from_class`

In fact, I could probably create a convenience method that does the third
and fourth steps together, something like `Parrot_api_pmc_new_from_array`.

One thing that throws a little wrench into my plans is that I need to support
HLL mapped types, even though the interpreter has no notion of a 'current'
HLL namespace. This means I would need to add in an extra parameter to any
API that created a specific PMC type to help inform decision of which type
to use. This is something that may have to wait until later, but it's worth
thinking about now.

By this weekend I would like to have all these pieces in place, and then we
will be able to work with PMC with relative ease from our embedding
applications. Next step is to exercise all these new API functions from
[Parrot#][parrotsharp], and create proper wrappings for various PMC types from
C#.

[parrotsharp]: http://github.com/Whiteknight/parrotsharp
