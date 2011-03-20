Yesterday I completed a refactor of the Rosella Test library, and today I am
posting some of the things that changed, how those changes affect tests
written with Rosella, and maybe talk about why I made some of these new
decisions.

## Overview

Previously you would write your Rosella-based tests like this (in NQP):

    Rosella::Test::test(MyTestClass);
    class MyTestClass is Rosella::Test::Testcase {
        method test_foo() {
            self.unimplemented("Don't be lazy, write the test!");
        }
    }

After the refactors, that same test would be written like this:

    Rosella::Test::test(MyTestClass);
    class MyTestClass {
        method foo() {
            $!context.unimplemented("Don't be lazy, write the test!");
        }
    }

This is a little bit less verbose and much more flexible. Here's a list of the
things that changed:

1. Test classes can be any class, you do not need to inherit from
   `Rosella::Test::Testcase` anymore. In fact, the Testcase class (with the
   lower-case 'c') does not exist.
2. Test methods do not need to be prefixed with "test_" anymore. By default,
   all methods in the target class will be used (with some infrastructural
   exceptions). You can still specify a prefix to use for tests, if you want.
3. To mark a test as todo, or unimplemented, or anything else, you now act
   through the `$!context` attribute, not through methods on the current test
   object.

These are just some of the visible changes. Now I'm going to talk about some
of the deeper ones.

## TestCase and TestContext

Before we had the `Rosella::Test::Testcase` class. When you created a test
class you inherited from Testcase. The test Suite would create one Testcase
object for every method in the class, and execute that method on that object.
By using separate objects for each test we gain insulation which helps to
protect one test method from sideeffects from another. However, this also
prevents us from sharing data between test methods, because each was executed
in a separate objec and no data was shared between them.

The problem with this approach is that we had methods on the Testcase class
itself which were used for controlling the test. These methods, such as
"__set_up()", "__tear_down()", and "todo()" caused a semi-predicate problem:
we had to have a way to separate out the methods which represented actual
tests from those which were used for internal purposes. This is why test
methods needed to be prefixed with "test_", to set the test methods apart.

In current Rosella, we replace the Testcase class with two new classes:
TestCase (with a capital "C") and TestContext.

TestCase is the test object on which the test methods are invoked. You do not
need to subclass TestCase. Instead, Rosella's Test library takes your test
class, extracs the method PMCs from it, and then executes those methods on an
instance of TestCase instead. This gives you increased test insulation, and
also avoids the semi-predicate problem: TestCase has no methods of its own, so
we don't need to worry about accidentally including a method which is for
internal use only.

The only thing that TestCase really provides of use is an attribute called
`$!context` (or just "context", if you're writing your test in a language like
Winxed which doesn't use sigils). The `$!context` attribute is an instance of
TestContext.

TestContext is a shared data item. A single TestContext is used for all tests
run in a single Suite. This gives us the ability to share data between tests
in a controlled way if we want, without polluting the actual TestCase
instances. TestContext contains a hash-like data store, some information
about the currently executing TestCase, and a handful of control methods to
tell what the result of the TestCase is.

## TestContext and User Data

## Configurability

## Future Work

Besides a few bug fixes, the Test library is basically in the form it will
take for the first Rosella release. When will that release be? I'm planning
to probably target Parrot 3.3 in April.
