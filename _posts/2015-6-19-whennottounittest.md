---
layout: post
categories: [Testing]
title: When Not to Unit Test
---

I used to be the kind of guy who cared about *test coverage*. You know, that
percentage value you get when you run your tests and count up how many lines
of your code were actually executed versus how many lines of code you have
total. Too little coverage causes your testing tool to spit out red warning
messages, flashing indicators and sad face emoticons. The goal for any system
is to reach 100%, no matter the cost.

What I've learned in the intervening years is that not all code is created
equal from the perspective of unit testing. Different types of code get tested
in different ways and some code doesn't need to get tested at all. Learning
when to test and how to do it when necessary, is a big step down the road to
software enlightenment.

Let's talk about a few different types of code, whether they are worth testing and,
if so, how to do it.

## External Dependencies

As a general rule, you do not need to test your external dependencies. That is,
you aren't testing code you didn't write. You need to trust that the original
authors did their due diligence. If you can't trust it, why would you be using
it in your project?

This is not to say that you should just use any old third party library sight unseen. This is not an excuse for you to shirk your due diligence. Put together a proof-of-concept, read user testimonials, and make sure the product you're getting is worth having. But, once you decide that you trust the makers of the library enough to use their code in your project, it's not your responsibility to unit-test their code. If their code has problems, you file bug reports. Let them fix and test it.

External dependencies are things like your platform standard libraries, your
database, your network, your operating system, referenced binaries, device
drivers, etc. Unless you're running custom builds of these things, don't bother
with "sanity" tests to prove that they work as advertised. If you are running a
custom build, you should have unit tests on that project, not in all the
upstream ones. It's a waste of your time to be testing code which you didn't
write, because the vendor has already done that.

Consider the case where I have some data access code that does a little bit of
data transformation, validation, and then stores the results into a database.
Since I don't need to test the database, I can inject a mock object or a stub
into my test to examine my stuff in isolation.

Consider also the case of a web app, running on a webserver, under some sort of
MVC framework. I don't need to test that the webserver accepts connections or
that my MVC framework correctly routes requests to my controller. All I need to
test is the actual logic which I've written to execute once the MVC framework
is done all its routing magic.

Consider this example controller method from ASP.NET MVC:

{% highlight csharp %}
public ActionResult DoWork() {
  // Complex business logic here
  return View();
}
{% endhighlight %}

To test our business logic in this configuration, we would need to instantiate
a controller, execute our action method, and then verify the returned view. The
majority of our test will be in testing components we didn't write! If our
business logic lived in a separate location, like a Service Layer or Domain
Model, we could test those things directly and leave the UI out of it.

Notice that I'm only talking about unit tests here. Integration tests, where you
would be testing your entire integrated application, are a different beast
entirely and shouldn't be skipped in cases like this.

## Internal Dependencies

Internal dependencies are things that are written in-house but which are
separate from the current project. If your internal dependencies are already
vigorously tested elsewhere, you don't need to test them again in your current
project.

In other words, you should test things in the appropriate places. You do need tests, but you don't
need or want them spread out all over the world. Put tests where they belong and trust that tests
exist and are passing when you attempt to reuse an existing library or product (if there are no
tests, or if tests exists and don't pass, that's a bigger cultural issue which needs immediate and
decisive resolution).

## Simple Gateways, Adaptors, Bridges, and Facades

Simple classes which provide access to other components but do not contain
any meaningful logic of their own do not need to be tested.

Consider the case where I am using EntityFramework as my ORM, and I write a
simple `IRepository<T>` wrapper type around the EntityFramework `DbSet<T>` type.
[As I've discussed elsewhere](/2015/03/21/repository.html), `DbSet<T>` *is an implementation of the Repository
pattern already*, so your wrapper is just an Adaptor to make EF fit into your
solution a little nicer (and also to provide an abstraction boundary, for
various tangential purposes, such as simplified mocking for tests).

When you have a situation like this, you don't need to test your
`IRepository<T>` type, because it's methods have a one-to-one relationship with
the underlying EntityFramework methods and because EntityFramework doesn't need to be tested by your
application. If EF is well tested, and if you've done
basic due diligence like checking that your `DbSet<T>` reference isn't null or
disposed, you can be certain that your `IRepository<T>` works correctly. Don't
waste time testing it if you know it's right because it's too simple to get wrong.

Look at this method:

{% highlight csharp %}
public void WidgetResult DoWidgetThing(WidgetArgs args)
{
  return _widgetMaster.DoWidgetThingInternal(args);
}
{% endhighlight %}

Under the dubious proposition that this code is worth having in the first place, it's obvious that
we don't require unit tests here. If `WidgetMaster.DoWidgetThingInternal` is already properly tested,
then what can possibly go wrong here? Well, `_widgetMaster` could be null, in which case this method
would throw a null ref exception. If `_widgetMaster` is being injected through a constructor
parameter and you know it's not null, that's a case that isn't worth considering.

Code which does nothing except make straight-forward, trivial calls to other methods which are
themselves well-tested, are likely not worth testing. Here's another, slightly more complicated
example lifted from an ASP.NET WebAPI setting:

{% highlight csharp %}
public void ThingResultDto GetThings(ThingRequestDto requestDto)
{
  DomainRequest request = _requestTranslator.TranslateToDomainRequest(requestDto);
  DomainResult result = _domainService.Handle(request);
  ThingResultDto resultDto = _resultTranslator.TranslateFromDomainResult(result);
  return resultDto;
}
{% endhighlight %}

This case is more complicated than before, but still very simple. We translate a user request from
the API into a domain request object, pass that off to some kind of service layer class, then
translate the domain result back into a DTO suitable for sending back over the wire. If
`RequestTranslator`, `DomainService` and `ResultTranslator` classes are already well tested, is
there any value in also testing this `GetThings` method in your WebAPI Controller? Integration tests
are always valuable, of course, but in this case unit tests seem superfluous. You don't *need* to
write a test here, because there is no non-trivial, untested logic. Method calls and variable
assignments are details of your language, compiler or runtime, and these things are third party
tools. You don't need to test third party tools, as I've already said.

I know you're probably thinking "But what if somebody else on my team changes this method? What if
the order of statements is rearranged, or if new logic is added to it"? To this I have a few replies:

1. You can't rearrange the order of statements in this method, because the type system and the
   compiler prevent that. You can't use the variables before they're defined, and you can't swap a
   `ThingRequestDto` in place for a `DomainResult`, so those kinds of changes are impossible.
2. By cultural convention, your team should know to keep thinks like MVC controllers thin, and move
   non-trivial logic into the Application Service Layer, or Domain Model, or wherever else. In that
   case, you shouldn't worry about people adding testable logic to places that are intentionally
   kept simple.
3. What happens normally, when you change logic in a method? You go to your test suite and ensure
   that either the tests continue to pass or that new tests are added to verify the new behavior.
   If you're adding non-trivial logic to a simple method which is not previously tested, normal
   operating procedure tells us that we need to add a test. I'm not saying some things should never
   be tested, I'm saying that some things in certain conditions are not worth testing. If something
   which hadn't been worth testing suddenly is worth testing, test it.

I'll talk about this in more detail below, but it's worth pointing out that the amount of tests
you need is proportional to the value of the code. If the code does nothing of value, and cannot
(according to the rules of your language, compiler and runtime) do anything other than what is
represented, you don't need tests.

(Notice that if you add custom logic in your adaptor, such as data mapping,
validation, consistency or other rules, you *will need to test that*).

## Orchestrators

Orchestrators are classes which take complex tasks, break them up into logical
subtasks, and delegate those subtasks to child objects for processing. This is classic Map-Reduce
and things like it: Map a large aggregate task out to several smaller tasks, and then reduce the
various result sets down into a single result.

Consider the case of a Service Layer which interacts with your Domain Model and persists
results using a Repository:

{% highlight csharp %}
public class FooService {
  private readonly IRepository<Foo> Repository;
  public FooService(IRepository<Foo> repo) { Repository = repo; }

  public void DoTheThing(ThingArgs args) {
    Foo foo = Repository.Load(args.FooId);
    if (foo == null)
      return;
    foo.Thing(args.Value);
    Repository.Update(foo);
  }
}
{% endhighlight %}

This class breaks down the high-level domain task "Do the thing" into individual bits: Load
the target object, perform an operation on it, and store the results of that operation back into the
database. Taken together, this all sounds complicated. But when you break the task down into
the individual operations and delegate those operations to the appropriate classes, it's not a big
task anymore.

What you should notice is that this `DoTheThingAndSave` method doesn't do much except call a
sequence of methods on other classes, just like our WebAPI Controller method above. "But wait", I
can hear you saying already, "There's an `if` statement in there! This method has testable logic!"
Not really. The `if` is a simple null ref check. Test it if you want to, but I think the value
proposition is dubious. Besides that `if`, this class doesn't really have any logic to it Besides
calling a sequence of methods in a required order. The compiler and type system guarantee that the
method can't be called in order, or that one of the calls be omitted without throwing an error about
using an uninitialized variable.

The loading and updating logic
happens in the repository, which your tests are going to mock anyway. The null
check is trivial, and then the rest of the method is a simple redirect to
`foo.Thing`. If `Foo.Thing()` is already tested, there's no reason to
test `FooService.DoTheThing()` also.

"But wait!" I can hear you yelling from over the interwebs "What if we add more logic to
`FooService` later and these trivial behaviors you mention become more complex or even get broken?"
You don't need tests so long as your code is trivial. If you lose that property, you do need to add
tests. But then, in the interest of being lazy, maybe we make sure that things which are trivial
remain so. This is a cultural effort, and one that your team would need to know and understand.
The easiest tests to write are the ones which don't get written at all. The fastest tests
to run are no tests. Make sure your team recognizes that keeping certain bits of logic trivial is in their
own best interests. Again, if something can't stay trivial, tests need to be added.

Our `FooService.DoTheThing()` method loads data out of our data store into a business object,
and then performs an operation on that business object. Where are we going to add complexity anyway?
We need some kind of precondition validation on the ID? We can put that in the repository. We need
another condition in there for an early exit on incomplete load? We can add a Specification object,
which will be separately tested. We need to change which method or which series of methods we call
on our `Foo` business object once it's been loaded? Maybe we refactor the body of this method out
into a handful of interchangable Strategy objects, which are individually tested. What if we need to
add more method calls at the end, after `foo.Thing()`? Well, we could compose many small
methods on `foo` into a single orchestrating method on that class, or we could just add them all
at the end. What will a test of `FooService.DoTheThing()` show us in this case? We'll set up
a mock repository to return a mock `Foo`, which will count method calls and verify ordering? We
start to venture dangerously far down the path of over-specification and micromanagement, which are
big reasons why mock objects get such a bad rap in some corners. We do not need tests to prove that
the order of methods called in our code is what it is in our code when our code works correctly.
See? it's even absurd to try and describe what we're trying to test!

If your try your absolute best and still can't find a way to keep this method simple in the face
of changing requirements, then by all means add tests for it. You don't need tests when things are
trivially simple, but when you lose simplicity you lose the benefit of not needing tests.

My point is this: our `FooService.DoTheThing()` method doesn't need to be tested and can
resist the need to be tested because all testable, non-trivial logic can and should be moved
elsewhere. This class is a simple orchestrator, delegating the real work out to other classes.

We absolutely do not need to test that our programming language can assign
a variable correctly, that it can correctly test for null, or that it can
correctly call a method. If we are indeed using mocks or stubs for our
Repository instance, we don't need to test that our mock object library can
correctly create mock objects, or that those mock objects return the values
we've told them to create. All these things are already under test, safely
filed away in our "External Dependencies" folder.

It's a heck of a lot less work to setup a unit test for a class which has no dependencies, few
dependencies or extremely simple dependencies. Classes which bring together a more complicated set
of dependencies, where Service Layer is a common example of such, are harder to create tests for
and therefore there is benefit in avoiding the need to test these things. If your orchestrator needs
to provide more behavior, you can encapsulate that in another method on an existing dependency or
in a new dependent type, and delegate out to that.

On a side note, I occasionally hear people arguing about whether the Service Layer should be thin or
thick. My money comes down on the side of being thin (and therefore trivial) for exactly these
reasons. A Service Layer, in my mind, exists to bring together dependencies (you *are* properly
using Dependency Inversion, right?) and delegate out tasks to them. Simpler classes are easier to
reason about, and being able to reason about things means it's easier to understand what they do and
feel comfortable that they do the right things.

On yet another side note, I see posts around the internet for "Are you a C expert?". These
posts invariably show code examples of things like weird instruction ordering, weird type-casting,
weird out-of-bounds issues and other weird code and then end with some little snip about how you
can't really call yourself an expert if you can't answer these questions about the language. In
retort, I like to point out that knowing not to write code like that is a much more valuable skill
than knowing exactly what bad things happen when you do. Not walking into the minefield in the first
place is a much stronger sign of intelligence in my mind than walking in with some heuristic for how
to avoid most of the mines. I'll gladly call you an expert if you know what code to not write, even
if you can't tell me exactly *why I shouldn't write it*. Think about this next time you're designing
code for testability. If your code is simple and follows best practices, you don't need a
super-expert to sort it out, and you don't need big complicated tests (or maybe any tests at all!).

After all that my thesis stands: Any class which does nothing but bring together dependencies and
delegate to them can be made trivial, and trivial classes like this don't need to be tested.

## Pure Data Objects

Any object which is pure data and has no logic or only trivial logic (null
checking constructor parameters) does not need to be tested. You need to trust
your programming language when it says it will make public `get`/`set` property
accessors work as advertised.

Things like DTO patterns and Null Object patterns fall into this category. If there is no logic,
there is no need for a test.

## What Do We Test?

So far I've listed several things which do not need to be tested. What's left?
What do we test?

Let's start by considering a tree. Tree has a trunk. Trunk splits into branches. Branches into twigs
and at the end of the twigs we have the fun, interesting stuff like leaves, flowers and fruit. The
inside of your tree is just wood: trunk and branches. Leaves, flowers and fruit form a shell or
canopy around the outside, where the action is. Leaves need to be on the outside of the structure to
receive unobscured sunlight. Flowers form on the outer edge where pollinators are likely to fly by.
Fruit forms where the flowers were. New growth, stems and shoots, typically come out where your
leaves are, with buds and free-flowing metabolic energy. The inside of your tree is the structural
stuff, the wood. The outer edge is where the interesting stuff happens and where new growth is
added.

Now let's think about your program call graph like a tree. Your trunk is your entry point. From
there you call methods on your Orchestrators (your branches) which in turn eventually call methods
on your testable classes (leaves, flowers, fruit).  (or, if you're using Dependency Inversion, which
you should be, you take references to child objects and call methods on them). The wood is just
boring and structural. It works because it must. If the wood doesn't work, there is no tree. If the
wood works but the leaves don't, what you have is a big waste of time and energy.

You start with one method on one object, which calls more methods on other
objects, which in turn calls even more methods on other objects, and so on and
so forth. At the very end of our graph, the leaves, we have things which either
call out to external tools or else do pure work and return the results. External tools are already
tested, so what's left for us to test is our own pure methods. Methods which we write, which
perform logic and return.

So think about your code as having the following three types of objects:

1. **Branch Classes** which do no work themselves, but instead delegate work to
   children, and then aggregate results back to the parent. (Orchestrators)
2. **External Gateways** which interact with external resources and libraries. These are basically
   orchestrators themselves, because they do no real work but instead reformat the request and
   send it on to the external resource like any other dependency.
3. **Pure Classes** which perform some kind of computation and return a result
   without calling external resources or only make trivial calls.

In an ideal project, if you've really taken the time to structure your code
in the way I'm suggesting, you only need to test your non-adaptor Pure Class leaves. Your
Branch Classes just delegate tasks to children so those are trivial and don't
need tests. Your external resources are no going to be included in your unit
tests so your External Gateways will probably be replaced with mocks or stubs
and won't be tested either. The only things you need to test are your Pure
Classes, which might not necessarily be "pure" in the sense of being completely
side-effect free, but they do work and return results without calling out to
another class.

If you can separate out your classes into these three piles, classes which only
delegate but do no real work, classes which are simply abstraction boundaries
around the outside world, and classes which only really do work and do not
delegate, you can focus your testing on the last group and be confident that
you do not need to worry about anything else.
