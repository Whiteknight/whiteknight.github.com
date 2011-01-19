I've talked on a number of occasions about problems with Parrot's exceptions subsystem. The system is functional and usable right now but really leaves a hell of a lot to be desired in terms of features and capability. One thing that jumps out immediately to me is that Exception and ExceptionHandler PMCs are not subclassable, creates a huge hassle for HLLs any libraries who want to be able to create and use subclasses with new behaviors.

I've started a new branch, `exceptions_subclass`, to work on this issue. I'm not going into it
blindly however; several months ago Parrot contributor Tene started working on the exact same problem. His branch has become too old and out of date at this point to continue the work there, so I started a new branch to follow his lead.

There are a few refactors that we really need to go through here, to tackle some important tasks. Here they are, in no particular order:

1. Make "Exception" a role, not a type. Parrot should be able to throw any PMC that "does Exception", including the Exception PMC, subclasses of the Exception PMC, and any other arbitrary type that implements the correct interface. This, at least in part, is the work I'm doing in my current branch.
    * As an addendum, we probably want to clean up the Exception interface, since it currently relies very heavily on named attribute access which isn't always very efficient. Exception PMCs should be very easy to work with.
2. Make "ExceptionHandler" a role, not a type, for all the same reasons as above.
3. Create multiple types of ExceptionHandler, some of which can be Sub-like, others can be like normal labels that we have now, and still others can be different things that I haven't even imagined yet. If I have an ExceptionHandler subclass which implements VTABLE_invoke, I should be able to do anything with it.


