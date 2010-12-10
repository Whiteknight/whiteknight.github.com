It's hard for me to think of another project that could have such an important
and immediate beneficial effect on Parrot as a new Metaobject Protocol (MOP).
As a simplified explanation, the MOP is the way the object system is
implemented. A good MOP can provide lots of cool features: Reflection and
introspection, runtime type definitions, runtime type compositions, dynamic
inheritance, etc. A bad MOP, like the one currently used in Parrot, can do a
lot to hurt the platform and makes it inattractive for developers to use.

And, since the MOP is involved in creating objects, managing objects, and
working with objects (including calling methods), it's implementation can have
a huge effect on system *performance*. That's right, I said the "P"-word. I'll
wait a minute so everybody's heartrates can get back down to normal.

Parrot needs a new MOP the way a plane need a pilot, or the way that a loaded
gun in a room full of well-sugared children needs a good trigger lock. If we
don't get it, something bad is going to happen. Hell, bad things have already
happened.
