Somebody asked me what the plan was for removing IMCC from libparrot. I
mentioned that there was no written plan, just a series of steps in my head.

Some ground work is [already done][imcc_cleanups]. From here, the next steps
are:


1. Continue cleaning and unifying the IMCC interface functions. We want to
   have a very small set of interface functions with non-overlapping
   responsibilities.
2. Implement a simple [PIR compiler PMC][pirpmc] to expose the new IMCC
   interface functions via a set of methods. These methods should try to
   follow PDD31 as closely as possible, at least until we propose serious
   changes to that document (which will happen in the coming months).
3. Detangle Parrot's interpreter from the `imcc_info_t` struct, so that we
   aren't keeping IMCC's state information in the interpreter. Instead, keep
   it in the new PIR compiler PMC for reference.
4. Implement a new interface for PackFile PMCs (be it a method or a new opcode
   or whatever) to manually execute `:load` and `:init`. This is necessary
   so we can replace the current functionality of the `load_bytecode` op. This
   is going to require a deprecation cycle to change.
5. Change all places in Parrot which call in to IMCC to use the new compiler
   PMC instead.
6. Do not register the PIR compreg compiler in libparrot itself, but instead
   register it in the front end.
7. Change the makefile so we do not build IMCC into libparrot. Compile it into
   its own shared library, and link it into the Parrot executable.

At this point, seven short steps from now, we will be able to build and
install libparrot without IMCC. It will still be readily available, but it
won't be available by default from libparrot.
