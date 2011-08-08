Here's a question we took a little bit of time today to talk about: What does
the ultimate, long-term debugging framework in Parrot look like? By the Parrot
5.0 or 6.0 release, or whenever, assuming we put some development resources
into it, what will our debugger look like? What will it be? How will it be
written? What kinds of actions will it perform and what kinds of features will
it have?

soh_cah_toa, one of our GSoC students this year, is working on a debugger for
Parrot called Honey Bee. `hbdb`, as it's called on the commandline, is a nice,
simple design, but the implementation of it has it snag after snag over the
summer and it's looking likely that come pencils-down, he won't have been
able to implement all the features he discussed on his proposal. Not that we
fault him for it. He's put in a Sisyphean effort over the summer, and we've
learned a hell of a lot from his work. HBDB will exist at the end of the
summer, and it will have some functionality that is worth looking at. Also,
soh_cah_toa and a few other hackers, including myself, are interested in
continuing the work after the summer is over. The most valuable part of this
project might not be the software he produces, but instead it very well might
be the lessons we've learned about what we need to do better before we can
really do debugging *right*.

What does doing debugging the right way look like? Well, it's fun to think
about now, and it's worth comitting some of the ideas to paper, in a manner
of speaking. Here are two things we know we need:

1. Significant improvements to our debugging data store. soh_cah_toa is
   talking about using something similar to the DWARF debugging format for
   PBC. Implementing that would be a great step, and would be very helpful
   for many purposes.
2. An improved API for reading various bits of debug information, and
   introspecting things from a packfile. In addition to debugging, this would
   be great for profiling and other analysis tools.

Those two things are the necessities for any debugger-related project we
undertake in the future, and the faster we can get started on them, even if
only in short fits and spurts, the better.

But beyond the common necessities, what do we want an eventual debugger to
look like? Knowing what we know now, and realizing we have a long time
horizon to accomplish it, what do we want to make?

Keeping in mind my current work with the
[new parrot frontend][parrot_in_parrot2], and work I've done recently on the
PackfileView PMC and cleaning up the packfile subsystem API, it should
probably not surprise anybody to learn that my conception of the future
debugger is written in PIR or an HLL and runs on top of Parrot itself. In
terms of pluggability, reusability, and flexibility, I can't think of a better
situation. Running bytecode is the perfect platform for executing and
analyzing bytecode and live data objects (PMCs and strings, etc). We have lots
of tools available now, and only a relatively scant handful of new features we
would need to add to make such a debugger possible.

We have an OpLib PMC which allows us to look up opcodes by name and by number.
We have an Opcode PMC type which allows us to introspect opcodes to get their
argument lists. We have Ptr PMC and MappedByteArray PMC and related types
which will allow us to read in a raw buffer of bytecode and examine it. This
is an important start. What we need to do is add in some kind of mechanism to
execute a single opcode with given arguments. Then, we can create a PIR loop
over the bytecode buffer, extract out the opcode, examine it to get the
arguments, then pull the arguments out of the bytecode and pass it to a
routine to execute just one op at a time. One method to write, and we
basically have a PBC runloop running on top of Parrot. That's the first step.

If we have a PMC type representing a location in bytecode, call it
CodePointer, we can have our HLL-based runloop create these objects,
instantiate them with the current code index, and pass them to callbacks for
analysis and control. The CodePointer type would take a `pc` index value, a
PackfileView PMC, a ParrotInterpreter PMC and maybe a CallContext PMC as its
only state. It would provide methods to use those values to look up other
interesting details: Current annotations (source file and line number),
current opcode, values of constants and variables at the time, register
values at the time, call frame information, etc. Maybe it could even contain a
continuation to automatically "rewind" execution back to a point of interest.

If we have these two pieces, creating a debugger, a profiler, or a variety
of other tools in PIR or an HLL becomes easy. Load in bytecode
