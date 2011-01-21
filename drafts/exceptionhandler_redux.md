When I invoke an exception in Parrot's PIR, the operation more-or-less looks
like this:

    .sub foo
        push_eh my_handler
        
        $P0 = new ['Exception']
        $P0["message"] = "You done goofed!"
        throw $P0
        ...
      my_handler:
        .get_results($P1)
        $S0 = $P1["message"]
        say $S0
    .end

tracing through this program is pretty helpful, even though it appears
straight-forward enough. The `push_eh` opcode pushes a new handler onto the
stack of handlers. An ExceptionHandler is a PMC type which is a subclass of
Continuation. When I call `push_eh` with a label value, I'm essentially
creating a new Continuation and setting it's target to the given label. When
I invoke the ExceptionHandler, I jump to that label.

I create a new Exception object, set up some data in it (in this case, just a
string message, but there are many other fields we could set), and `throw` it.

Internally, this is where things get a little bit interesting. The exceptions
subsystem calls into the scheduler to get a list of available handlers. The
scheduler in turn asks the current Parrot context for a list of active
handlers, then iterates over each until it finds one which is capable of
handling the current exception. How do we know if a handler can handle a
given Exception? By calling the `"can_handle"` method on each handler. Simply
throwing an uncommon exception in a system with multiple active handlers can
generate dozens of method calls. As anybody will tell you, method calls in
Parrot are currently not cheap.

At this point, we set up a normal Parrot Calling Conventions (PCC) invocation
with signature "P->" (one argument, the exception, and no expected return
values) and invoke the ExceptionHandler continuation. In a system with one
active handler, it takes 2 PCC invokations to throw a single exception, but
we don't have hardly any flexibility with this system. We get one Exception
object which, in the basic case, gets one payload field. We can also subclass
Exception to add more stuff to it, but the Exception role is pretty complex
and any errors in implementing it will probably cause Parrot to crash.

This is all not to mention that to set up an ExceptionHandler to receive only
a single type of Exception requires *at least* one additional method call to
pass in the necessary type constraints. That's three method calls if you're
doing anything even remotely fancy. And by "fancy", I mean "normal".

We can do better. Today I'm going to brain-storm some ideas about how.

If Exceptions are properly subclassable, and if ExceptionHandlers are just
continuations and are invoked through PCC, we can use Parrot's powerful
multidispatch system to dispatch them.

If ExceptionHandlers are properly subclassable, we can define subclasses of
Exception which always take certain subclasses of Exception. This reduces the
need to specify the handleable types on every single handler that we throw.
We simply create the correct subclass of ExceptionHandler and push it. The
subclass already knows what types it handles, so we don't need to tell it.

Now here's an idea for a bigger change to the interface itself. What if we
allowed exception handlers to be invoked like any other invokable, from PIR
with `()` parenthesis? In other words, instead of telling the system to find
its own handlers, what if the user had a hand in it? Here are some example
snippets of code to illustrate what we could have:

    # First, we can get the currently active handler multi from the context
    handler = context.'get_active_handler'()
    handler(exception)  # Might be a multisub or new "MultiHandler"
    
    # Second, in lieu of multihandlers, we ask the context for a matching
    # Handler:
    handler = context.'get_suitable_handler'(exception')
    handler(exception)
    
    # Third, if we don't want to do things manually, we can tell the Exception
    # to just do whatever it wants. This emulates current behavior of the
    # throw opcode, without the opcode. Internally, this probably does the
    # same as code snippet 1 or 2 above
    exception.'throw'()
    
Here's where things start to get pretty interesting: What if we didn't
restrict exception handlers to only taking a single argument? What if they
were like any other invocable and could take any number of arguments?

    exception.'throw'(args)

If we did this, we could separate out the payload from the exception, and
pass the two individually. Further, we could pass more than a single object
as a payload.

    exception.'throw'(arg1, arg2, arg3, ...)
    
What we start to do here is separate out the Exception object, which becomes a
slimmed-down cache for some bookkeeping information, from the actual payload
object of the exception itself. The payload, which can be any object we want,
can be "thrown" as if it were an Exception, but without actually having to be
one.

    $P0 = new ["MyArbitraryObject"]
    throw $P0, "Oops! It broked!"

On the receiving end, the handler will receive the Exception (created and
populated internally with the stacktrace and some other details) and the
payload separately:

    my_handler:
       .param pmc exception
       .param pmc payload
       ....
       
As an aside, we did recently close a ticket as WONTFIX that suggested using
`.param` notation for exception handler arguments instead of `.get_results()`.
Without a system where exception handlers can take multiple parameters, such
a ticket was unnecesary. If we do completely revamp our exceptions system,
maybe it needs to be reconsidered.

By completely separating out the payload from the Exception, and changing
ExceptionHandlers to being normal, first-class invokables, we can start to
slim down the Exception PMC so it becomes smaller, cheaper, and has a simpler
interface to subclass.
    
The most compelling thing for me is this: If Exceptions are thrown by a
method call:

    exception.'throw_to'(handler, args, ...)
    
    # or
    
    handler.'catch'(exception, args, ...)

Suddenly the entire exceptions subsystem becomes almost completely
subclassable. Any HLL can completely rewrite it if they choose to, to support
different semantics. If I want to implement a `finally{}` kind of block in my
`try`/`catch` sequences, I can subclass ExceptionHandler and add that in. If
I want to catch exception from a C embedding application with an NCI-based
handler, I subclass ExceptionHandler and add in an invoke override that
redispatches to the NCI handler routine. If I had a parser, I could throw
an exception to a subclass of an ExceptionHandler which was actually a
coroutine. Then, I could continue parsing up to a certain preset limited
number of parse errors before I finally gave up. The coroutine handler would
resolve any parse conflicts and print out an error. At 20 errors, we print out
a message saying "too many errors, aborting", and then the coroutine would
pass off to a different handler, not return back to the parser.

What I like about some of these ideas is that we gain a bunch of additional
features and flexibilities by actually *removing* a hell of a lot of code.
If ExceptionHandler is just a normal invokable, we don't need all sorts of
special code to set it up and invoke it: We just get a reference to it and
invoke it. We can cut out all the logic for Exception type ID numbers, we can
cut out all the special case "can I handle this" code that ExceptionHandlers
need to implement, and just do all our exception dispatching based on existing
multidispatch semantics of Parrot. I suspect we can cut the scheduler
completely out of the process, and just ask the context directly for a
suitable handler when we want one.
    
In closing, here's a short code example of what I'm talking about. Keep in
mind that this is all just rough brainstorming, I'm sure if we want to move
forward with some of these ideas that we can improve the look of the interface
significantly.

    .sub main :main
        $P0 = get_class ["Exception"]
        $P1 = subclass $P0, "FooException"
        $P2 = subclass $P0, "BarException"
        
        $P0 = get_class ["ExceptionHandler"]
        $P1 = subclass $P0, "FooHandler"
        $P2 = subclass $P0, "BarHandler"
        
        $P0 = new "FooHandler"
        push_eh $P0
        $P0 = new "BarHandler"
        push_eh $P0
        
        $P0 = new "FooException"
        $P1 = get_context_multi_handler
        $P1.'catch'($P0, "payload!") # the catch method is overridden in subclasses
        # default behavior returns us here, but we could also invoke a
        # continuation and go elsewhere
    .end
  
    .namespace ["FooHandler"]
    
    .sub 'catch' :method :multi(FooException,_)
        .param pmc exception
        .param pmc payload
        say "Caught a Foo with a payload!"
    .end
    
    .sub 'catch' :method :multi(FooException)
        .param pmc exception
        say "Caught a Foo!"
        $P0 = getattribute self, "continuation"
        $P0()  # my_handler
    .end
    
    .namespace ["BarHandler"]
    
    .sub 'catch' :method :multi(BarException)
        .param pmc exception
        say "Caught a Bar!"
        exit 1  # Bar is really bad. Just exit
    .end

This all requires a hell of a lot more thought, but I think there is some
gold to be mined here.
   

 