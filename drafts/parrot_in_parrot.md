Several days ago I wrote a post about writing a new
[JavaScript in JavaScript][jsinjs] compiler for Parrot. With that idea in the
back of my head and with some of the recent [IMCC][imcc_cleanups] and
[Packfile][packfile_tasklist] changes that I've been working on recently, I
started formulating an interesting idea about what to do with Parrot's
frontend in the coming months.

[jsinjs]: http://whiteknight.github.com/2010/12/07/javascript_on_parrot_plan.html
[imcc_cleanups]:
[packfile_tasklist]:

For the basic PIR compiler frontend (better known as the current Parrot
executable), with the task of compiling and executing a program written in PIR
we have a few basic tasks that we need to do:

1. Create the interpreter. This includes some basic command-line parsing for a
   handful of parameters that must be set during interpreter initialization.
   Specifically `--hash-seed`, `--gc`, and `--gc-threshold`
2. Parse the rest of the flags, separating out the arguments into three
   sets: The settings for the interpreter itself (warning and debug flags to
   activate, a few other settings) the settings for IMCC (the format of the
   input and output files, if any), and the arguments to pass to the PIR
   program.
3. Invoke IMCC to parse the input file into a proper Packfile, passing it
   any relevant commandline arguments.
4. Wrap the arguments for the PIR program up into an array PMC
5. Execute the program with the wrapped argument PMC
6. Destroy the interpreter and exit the program.

This is a pretty simple list of things. My idea starts off with an equally
simple premise: What if we did less startup work in C, and jumped into a
quick PIR "prefix" to do the rest? We would need to make a few modifications
to IMCC, maybe add a function or two to the embedding API, and change a few
things in the makefile, but it shouldn't be too hard. The sequence would look
like this:

In C:
1. Create the interpreter, parsing the necessary subset of commandline
   arguments
2. Wrap up the commandline arguments into an array PMC
3. Get a reference to the prefix bytecode (compiled in to the binary, like a
   pbc_to_exe program), load it into the interpreter and jump directly to it.

In PIR:
4. Create a new [PIR compiler PMC][pirpmc]. Register it with `compreg`.
5. Parse out the remaining commandline arguments, separating out the ones that
   go to the PIR program, and using the rest to set parameters on the interp
   and the PIR compiler PMC.
6. Use the new PIR compiler PMC to compile the executable into a PackFile PMC.
7. Get a reference to the `:main` function from the packfile and execute it,
   passing in any necessary arguments
8. Do any necessary cleanup and exit the prefix program.

In C Again:
9. Destory the interpreter and exit the program.

[pirpmc]: http://whiteknight.github.com/2011/01/14/exception_backtraces.html

This doesn't necessarily look any simpler, but the reality is that we end up
with much cleaner code. All the code in PIR for instance can be wrapped in
a single exception handler, instead of having to check the output of every
single C API call for success. PIR is also the most natural place to be
creating PMCs like the PIR compiler PMC, registering it, and calling methods
on it. Plus, we really get a jump on the idea of rewriting portions of Parrot
in Lorito.

Here is a short example of what a basic entry-point routine in Parrot would
look like, after some of the proposed changes to the PIR compiler and
[Exception PMC backtrace improvements][backtraces]:

    .sub __parrot_entry_point :anon :main
        .param pmc args

        push_eh __global_ex_handler

        .local pmc pir_compiler
        pir_compiler = new ['PIRCompiler']
        compreg "PIR", pir_compiler

        .local pmc compiler_args
        .local pmc program_args
        .local string program_file
        (program_file, compiler_args, program_args) = '__parse_args'(args)
        pir_compiler.'set_arguments'(compiler_args)

        .local pmc packfile
        .local pmc program_main
        packfile = pir_compiler.'compile_file'(program_file)
        packfile.'run_init_functions'()
        program_main = packfile.'get_main'()
        program_main(program_args)
        exit 0

      __global_ex_handler:
        .local pmc exception
        .local int exit_code
        .get_results(exception)
        finalize exception
        pop_eh
        __print_exception_backtrace(exception)
        exit_code = __get_exception_exit_code(exception)
        exit exit_code
    .end

This code is pretty straight-foward, though it would get a little bit more
complicated if we wanted to support additional options such as the `-o`
commandline argument, which compiles the program to a .pbc file and writes it
out but does not execute it, or the `-r` option which compiles to an output
.pbc file like `-o`, but then immediately reads in from that .pbc file and
executes it. However, even with these options added I suspect a prefix program
like this would be less than 500 lines of well-written PIR.

Notice also that to create a new executable for a different compiler, like
Rakudo, we would only need to make a relatively small number of changes to the
logic above (Load in the Rakudo library, register the Rakudo compiler instead
of the PIR compiler, etc).
