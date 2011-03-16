[Design by Contract][design_by_contract] programming is a style of development
characterized by setting up formal requirements on several areas of your
software and developing software so that those requirements are and remain
satisfied.

In the world of C, a basic way of doing this is to sprinkle calls to the
`assert()` standard library routine into your code. During development those
assertion statements check certain conditions in your code and cause program
failure if they are violated. Once the software is deployed, those assertions
can be turned off to save on performance overhead.

Design by Contract takes that idea to a new level by adding in the concept of
full contracts. A contract may specify several behavior and requirements, such
as pre- and post-conditions on method calls for objects (to verify the sanity
of the input arguments and of returned values), or to verify class invariants
(values which either must not change, or must continuously satisfy certain
critieria). If an invariant becomes unsatisfied there is evidence of data
corruption or, at the very least, object misuse. If a pre-condition on a
method call fails, an object consumer is calling the method incorrectly. If a
post-condition fails, the object is not returning the kinds of values it
claims to.

## Rosella Contract library

In [Rosella][] I've added a new Contract library to help explore some of
these ideas and maybe bring parts of them into the Parrot ecosystem. The
Contract library provides tools needed to use contracts in your code, but
doesn't provide the kinds of declarative contract syntax or static code
analysis features that other tools may provide. Instead, the Contracts
library intends to provide the features that those other more advanced tools
would use.

At the basic level, the Contract library provides an `assert()` routine which
can be used to test basic assertions in your code:

    Rosella::Contract::assert(1 == 5, "Madness!");

The assertion does nothing if the boolean first argument is true, it throws
an assertion failure exception with the given message otherwise. Assertions
must be *turned on*, or else they typically do nothing. Inserting the code
above into your software won't cause a failure by default. You need to turn
on the library first:

    Rosella::Contract::active(1);
    Rosella::Contract::assert(0, "whoops!");

Notice that this only turns of the logic inside the library, it doesn't
cause the code to disappear from the software entirely like a
preprocessor-based solution would (in C, for example). You have to use the
features of your particular HLL if you really want to make the contract calls
go away entirely in your deployment code.

## Contract Interfaces

Statically-typed languages like Java or C# provide interfaces which can be
used to specify what kinds of functionality a type provides without making
any assumptions or requirements about the actual implementation. In those
languages, objects can be passed around and referred to by interface reference
only, which helps to insulate object consumer code from the particular
implementations and subtypes used. A passed object can be of any custom type
so long as it satisfies the interface.

In dynamic languages where we don't have strong typing or where we don't have
a notion of an interface, this leaves a bit of a disparity. How do I know in
my library code that a person is calling me with valid objects? I don't want
to require people to subclass my existing classes and does/roles support in
Parrot is neither as functional nor as widely used as I would like. I would
prefer to keep things flexible and allow people to pass in any object so long
as it satisfies the requirements. To prove this, I can use the Interface class
in the Contract library to prove that a given object satisfies a necessary
interface:

    my $interface := Rosella::build(Rosella::Contract::Interface,
            My::Base::Type);
    $interface.verify_object($foo);

If the variable `$foo` provides all the publically-visible methods of
`My::Base::Type`, nothing happens. If not, we throw an assertion failure.
Again, if the contract library is disabled, this code does nothing no matter
what.

At the moment, the `Rosella::Contract::Interface` mechanism only verifies the
existance of methods by name. In the future it may have other capabilities
as well, such as verifying the existance of certain attributes by name (for
languages which allow public use of attributes).

I have a class object which is already written and makes certain assumptions
about how it is going to be called. I don't necessarily want to add in a bunch
of expensive checks on my arguments to the code, or litter my library with
assertion calls. Instead, what I can do is inject pre- and post-conditions
into the method itself without affecting the already-written library in any
way. To do this, I can use the `Rosella::Contract::Method` mechanism:

    my $method := Rosella::build(Rosella::Contract::Method,
        My::Base::Type);
    my %preconditions := {};
    %preconditions{"a_not_five"} := sub($a) {
        Rosella::Contract::assert($a != 5);
    };
    my %postconditions := {};
    %postconditions{"result_is_not_ten"} := sub($result) {
        Rosella::Contract::assert($result != 10);
    }
    $method.inject("MyTestMethod", );

    my $test := My::Base::Type.new();
    $test.MyTestMethod(5);      # oopsies!

After the injection of these conditions, any attempt to call this method with
the value 5, or any attempt to return the value 10 from that method will
result in an assertion failure. Again, if you set the Contract library to
inactive, these checks do not happen. For reference if you do fail the
assertion, here's the message you see on the console:

    In precondition 'a_not_five'
    Failed precondition 'a_not_five' for method 'MyTestMethod'

You set up the contracts and inject conditions into methods ahead of time, and
can get full run-time value checking without having to modify your program
code at all. Or, you can combine this method with various assertions and
interface tests sprinkled throughout, and turn them all off when you build
and deploy a release.

Or, instead of turning off the library internally, you could use a feature of
your particular HLL to completely filter these kinds of statements out of your
source code before compilation so they don't even appear in the generated
bytecode.

## Future

This is a quick overview of the new Rosella Contracts library I've been
playing around with. It doesn't have a hell of a lot of features right now,
but it's still cool to think about all the new diagnostics tools we can use
and we can develop in the future to help us with our code.

One feature on the immediate roadmap is the ability to add invariant
conditions to attributes, to verify that they are never set to values outside
of a predefined range.

I have a few other ideas floating around in my head too, but I would really
love to hear from potential users of this library which kinds of features
would make the most impact and what people would like to see added to it.


