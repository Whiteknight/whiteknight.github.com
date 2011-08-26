Security is something which is important for Parrot tackle, but which it
absolutely does not deal with now. There are a variety of reasons for this,
but I will cite two in particular: A lack of a design for a security system,
because of a lack of specific use-cases from users.

Today we had a discussion on IRC about security, where we talked about several
things. I asked for specific examples of what people wanted security for, but
I didn't get a whole lot of actionable information. Here's one reply I got
from Rakudo hacker moritz:

    <moritz> I'd like to be able to tell rakudo that it's not allowed to read
    files except from it's installation directory

It does seem like a simple enough request, but notice that it really doesn't
contain enough information for us to really design a security system. I mean,
it's trivial for us to have some kind of restriction list on the interpreter
to verify that in order for Parrot to interact with a file it either must be
a child of a path in a white list or must not be a child of a path in the
black list. Of course, if we have a way for a well-intentioned user to set up
that restriction list, it would be possible for a malicious user to simply
unset it before attempting an IO operation, so we're basically back to where
we started.

For any kind of individual security task we want to tackle, we really are
going to need some kind of over-arching security system to protect it. If we
don't protect the protection mechanisms, anybody can perform any operation
by simply undoing the protection set up around that operation. Even though it
seems like it should be simple to secure Parrot IO, if we really want it to
be "secure" by any definition of the word, we are going to need to think about
the bigger picture. Unfortunately, that's where things start to get fuzzy.

Lots of people mean different things by security. The top-level questions are
these: What are we securing? From what? From whom? Who does the securing, and
what makes that person so special? I would be surprised if many of these
questions had clear answers.

Let's look at moritz's request for the Parrot IO system, and work out a few
important questions that need to be answered for that system alone before we
can really start to implement a solution:

1. At what level should security settings happen? Do we set them at the binary
   level (in a C embedder application, before the first interp is created), at
   the interp level (where an unrestricted parent interp can set settings for
   the child), or at the context level (where a caller context can specify
   security settings for the callee and all subsequent points in the call
   chain). All of these have their own benefits and drawbacks.
2. Who sets the restrictions, regardless of the level where they are set? Do
   we require a system admin/root/sudoer user to change settings? If so, must
   we run the program as sudo every time, so we can set up the initial
   restrictions? Or, do we only need permissions to lift security
   restrictions, but not to set them? Or, can anybody set security settings on
   a child sandbox, but the settings of the sandbox cannot be modified from
   within itself? Or, once created, the settings of the sandbox are read-only?
3. What exactly do we mean by "read files"? We're going to need restrictions
   in FileHandle PMC's "`.open()`" method, certainly. What about other things
   like the "`.include`" or "`.loadlib`" PIR directives? What about the
   `load_bytecode` opcode? The `stat` dynop? the OS PMC `.readdir` method?
4. Several of those things I mentioned in question #2 above can be at least
   partially restricted by modifying the interpreter's list of search paths.
   Should those be restricted too, or should path restrictions happen at a
   different point and we ignore the search paths?
5. We can use NCI to get access to functions from the C standard library,
   including `fopen`, etc. When we disable IO, should we automatically
   disable NCI? Notice that similar arguments can be made for all subsystems,
   so does NCI just get shut off any time we implement any security
   restrictions?
6. It would be trivially easy to write a program in a whitelisted location
   that interacts with a file in a blacklisted location, then spawn a new
   program to do that. So, like with the NCI argument above, if we're in any
   kind of security situation must we also restrict things like the `spawnw`
   and `system` opcodes, and the pipe mode of the FileHandle PMC?
7. Is there anything we can possibly do about custom C-level extension code,
   such as dynpmcs and dynops? Should we just ignore that? Provide an API
   and expect extension writers to use it properly?

These are honest questions that we are going to need real answers for if we
want to just implement the simple requirement of preventing file accessess
outside of a particular directory. Some of these things I could suggest my own
answers to ("The interp level", "anybody, and settings are read-only", "yes to
all", "ignore", "restrict NCI in any security situation", "same with spawnw,
system, and pipes", "no"), but I don't really know what exactly our users
want. I hardly qualify as an ordinary, outside user of Parrot, so my answers
to these questions are basically just shots in the dark combined with some
intuition about how things are going to work once we start implementing. To
make real progress on the issue of security, we need real answers from real
users, and I can't just make that up by myself, or ask any of the other Parrot
developers to make it up either.

Notice also that while some of the questions above are general, they don't
cover all things that fall under the umbrella of security, and they don't
even get to some of the really hard questions about implementation,
interaction with OS-level permissions, etc.
