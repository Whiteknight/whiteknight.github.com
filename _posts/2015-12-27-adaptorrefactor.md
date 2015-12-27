---
layout: post
categories: [Architecture, Design, Testing]
title: Big Ball of Mud Refactor to Testability
---

We had a small utility application which took in messages off the service bus
and imported the data into ElasticSearch for querying. The task I was given,
relatively fuzzy at that, was to improve the application. In doing so, it was
expected that I would come up with some patterns and eventually some guidance
for how to employ and properly utilize ElasticSearch in the future.

Consider this something of a case study of my thesis from my previous post on unit testing. If we
set up things properly with unit testing in mind, we can focus our attention on classes which need
to be tested and can avoid testing things which do not.

The application was a classic ball of mud design. JSON was read off the service
bus, Newtownsoft.Json was used to deserialize the JSON into a `dynamic` object.
Several large methods searched for various fields on this object, massaged or
moved them (for search/faceting purposes) and then the whole thing was dumped
into ElasticSearch. All the important parts were implemented in basically three
large, unruly classes. I can't show any of the code here, but suffice it to say that this was the
land that SOLID forgot.

Here's a quick overview of the steps I used to start to transform this application into a reasonable
one which we can apply tests to and start to make guarantees about.

1. Use refactoring tools to clean the code without changing any behavior.
2. Encapsulate calls to external resources into dedicated adaptor classes.
3. Create a proper, strongly-typed Domain Model.
4. Separate like concepts into dedicated utility classes.
5. Create service layers to map incoming commands to methods on our Domain and Utilities.
6. Add unit tests and integration tests.

## Why Not Test First?

People familiar with TDD or who are very pro-testing might suggest that we write tests first so
that we can know what the behavior *is*, so that when we make changes we have assurance that the
behavior has not changed. The problem is that with a Big Ball of Mud system, there typically aren't
the kinds of abstraction boundaries or clean interface that make testing work correctly. You end up
either locking the old, bad interfaces in place with premature unit tests or else you end up having
to re-write your tests when you improve the interface. If the test is changing along with your code,
there's no real benefit to testing the old version of the code in the first place. That is, if the
test is changing, you don't have any assurance that the overall behavior of the system didn't
change. You're better off just making your changes, doing some ad hoc manual testing to prove that
things are the way you want them, and then adding your unit tests to prove the point.

## Refactor

I use Resharper, but any refactoring tools will do. What we want to do in this first step is to
clean the code making certain that we don't change behavior. We want to make it easier to read and
understand, because we can't do anything else if we can't understand it. Here are some examples of
things you can do to improve the quality of code quickly with a refactoring tool and some simple
editing:

1. Invert conditionals to decrease nesting.
2. Extract cohesive chunks of code into new methods with descriptive names.
3. Rename existing methods, properties, fields and variables to more clearly describe what is
   happening.
4. Remove dead code
5. Remove misleading documentation (you can add good documentation, but getting rid of bad
   documentation is more important in my opinion).
6. Add some comments. Bits of code which implement a single idea, but maybe aren't big enough or
   cohesive enough to turn into a separate method can get a comment saying what is happening. These
   comments then form an outline or checklist of what the method needs to do.

If you are interested in learning more about common and important refactoring techniques which can
help to make code more readable, understandable and maintainable, get yourself a copy of Martin
Fowler's seminal "Refactoring".

When you're done your basic cleanup, if you've done it correctly, the program should continue to run
like normal with no changes in behavior.

## Encapsulate External Resources

For this application, we had three external resources to interact with: a service bus to receive
commands from, ElasticSearch (via Nest) and a database for looking up some values and updating our
domain model before indexing into ElasticSearch.

The bus was serviced by a `Consumer` class, which read messages off the bus, parsed the command,
and dispatched the request to the appropriate methods in our ball of mud. It looked something like
this:

{% highlight csharp %}
public class BusMessageConsumer {
  public void ConsumeMessage(MyCommand cmd) {
    // Boilerplate setup here
    try {
      switch (cmd.CommandType) {
        case CommandType.DoTheThing:
          ...
        default:
          // log that we have an unrecognized command
      }
    } catch (Exception e) {
      // Error logging, error-handling, error-recovery, boilerplate
      return;
    }
    // Success logging, statistics, timing, etc
  }
}
{% endhighlight %}

Switch statements in code are often a sign that we can use polymorphism instead (Again, read
"Refactoring"). We can switch
this to use a Command pattern (which we will call a "Handler" here to not confict with the Command
Message pattern coming in off the bus), and move the switch statement into a HandlerFactory. Now
we have something like this:

{% highlight csharp %}
public class BusMessageConsumer {
  public void ConsumeMessage(MyCommand cmd) {
    // Boilerplate setup here
    try {
      Handler handler = new HandlerFactory().CreateForCommand(cmd);
      handler.Handle(cmd);
    } catch (Exception e) {
      // Error logging, error-handling, error-recovery, boilerplate
      return;
    }
    // Success logging, statistics, timing, etc
  }
}
{% endhighlight %}

Inside our `HandlerFactory` we can replace our switch block with a lookup table to respect the
Open/Closed Principle:

{% highlight csharp %}
public class HandlerFactory {
  public static Dictionary<CommandType, Type> _types = InitializeTypes();
  public static Dictionary<CommandType, Type> InitializeTypes() {
    return new Dictionary<CommandType, Type> {
      // default type registrations here
    }
  }
  public static void AddNewHandlerType(CommandType cmdType, Type handlerType) {
    // Validate that it's the correct kind of thing
    _types.Add(cmdType, handlerType);
  }

  public Handler CreateForCommand(MyCommand cmd) {
    if (!_types.ContainsKey(cmd.CommandType))
      return new NullHandler(); // Null Object Pattern
    return Activator.CreateInstance(_types[cmd.CommandType]);
  }
}
{% endhighlight %}

Now the front of our application is neatly separated from the rest of it. With a small amount of
additional refactoring to use dependency injection in our `BusMessageConsumer`, we could test that
messages with different command types do indeed go into the `HandlerFactory`, which returns a
`Handler` object of the correct type. But if you take a closer look, you'll see that our Consumer
class doesn't really do any work: It calls a method on `HandlerFactory` with the argument object
unmolested, and it then calls a method on the return value with the argument, again unmolested.
There's no real point to doing deep-testing here, so we can skip it for now.

The `Handler` objects represent a mapper from our request domain (`MyCommand`) into our
problem domain, and it's here where we want to start unit-testing in earnest. I added tests that
`HandlerFactory` returns the correct `Handler` subtypes given different inputs, and I started
planning tests for the various `Handler` subclasses as well. We aren't ready to actually test
those `Handler`s just yet though, we still need to decouple the other side of the business logic
from the data stores (ElasticSearch and the DB).

The database can be wrapped up in your choice of abstraction. I choose to use a simple gateway for
queries, but you could use a `Repository` or `Active Record` or any other data access strategy that
you think would fit. The book "Patterns of Enterprise Application Architecture", again by Martin
Fowler (I love that guy!) discusses these and more options in some detail.

ElasticSearch is written in Java, so if you are writing Java code you can spool up an embedded
instance of it for testing and not worry so much about encapsulating it. In C#, not so much. It is
far too onerous to spool up a fresh ElasticSearch instance for every test or even every test
session, especially when your CI routine is trying to do this on some remote build server in a
brain-dead automated way. That is, you can writing an Integration Test suite which does this, but
that is likely going to require help and support from your Infrastructure team or DevOps or
however your origanization delineates that kind of work, and it might not be worth your effort.
I think it's far better in this case to encapsulate ElasticSearch out
behind an air-tight interface and trust that Nest and ElasticSearch are doing what they are
advertised to do (A certain amount of testing, be it manual or automated, should definitely be done
at first and at any time that the version of these tools are upgraded to verify that they do,
indeed, do what you expect).

Notice that, taking this approach, we are limiting our use of certain features. Nest has features
where you can generate your own JSON, or generate your own identifier path strings, or even generate
some strings of commands in the Elastic DSL. If we are trying to rely on our abstractions and use
things like compile-time type checks and externally-tested code to make our case, we can't break
that encapsulation with hand-rolled strings of commands that we aren't providing ourselves with a
way to test.

One thing I've found over the years is that a Builder pattern works very well to abstract the
query-building interfaces of various data stores and ORMs. Using a Builder, you can still have an
air-tight abstraction but the abstraction can grow over time as you need and each new addition to
the Builder API gets a readable, descriptive name of what it is trying to accomplish. Consider an
interface like this, filling in the details of your particular ORM or storage technology:

{% highlight csharp %}
SearchQuery query = new SearchQueryBuilder()
  .MatchingSearchTerm(term)
  .WithVisibilityFor(CurrentUser)
  .WithType(type, subtype)
  ....
  .Build();

SearchResult result = query.Execute();
{% endhighlight %}

Using a Builder pattern like this to make an extensible abstraction over your data storage
technology (this only works for queries, you'll need a different strategy for INSERT/UPDATE/DELETE,
like a Command pattern), and setting up your system to make proper use of DI will allow you to mock
out your store and finally start isolating your business logic for testing purposes.

## Create a Domain Model

The `dynamic` type was added in .NET 4.0 with VisualStudio 2010. It gives us some of the flexibility
enjoyed by dynamically-typed programming languages and allows enough flexibility to put together
prototype code very quickly.

The code we had in this application looked something like this:

{% highlight csharp %}
dynamic model = Newtownsoft.Json.JsonConvert.Deserialize(jsonString);
{ %endhighlight %}

While this is decently fast for making prototype code, over time the strain on the system became
huge. The reasons are two-fold. First, because there was no documentation anywhere about what was
and was not supposed to be in the `dynamic` object. This lead to huge amounts of data being indexed
into ElasticSearch which didn't need to be there, because we didn't have an inventory of the fields
we wanted and any kind of mechanism to exclude fields we didn't want. The JSON was turned into a
`dynamic` and whatever was in that object is what ended up in the index. Notice that a DTO-style
object has some built-in "documentation" in the form of field names and types. Just this small
amount of extra structure would be head-and-shoulders above where we were, even if no other
comments or documentation was added ("self-documenting" code rarely has all the information you
want, but sometimes can have the basics which you *need*). Second, because there was no obvious and
enforced structure to the data, there was no obvious structure to the code which worked on that
data. The system had an organically-grown set of methods which had grown absurdly large, with loops
inside conditionals inside loops. The data being `dynamic` and there being no obvious rules or
expectations means that every property access needed to have a null check followed by a check that
the data was in the correct type. Much of this validation work could have been handed off to the
parser, if the original developers had taken the time to give the parser a little bit of the
type metadata it needed to do the job.

We actually need two objects here: One representing the JSON request coming in from the Bus, and
the other representing the storage type that we index into ElasticSearch. Once we have these two
model types, we can create a mapper object to convert from one to the other. This is the heart of
our system.

With a proper Domain Model in place, even if it was a little bit anemic in this case, we can start
doing some of our real testing: Test that the JSON messages coming in off the bus deserialize
properly into our Request Model. Test validation routines on our Request Models. Test that the
Mapper properly maps various Request Model objects into Index Models. Test that our data layer
properly hands our Index models off to ElasticSearch (or, test that we get all the way down to
our mock that is standing in for ElasticSearch).


## All Together

When you're starting out with a piece of software which is a little bit more sane than what I had
to deal with, the steps you use to make improvements might look something like this:

1) Test, baseline
2) Cleanup and Refactor
3) Test
4) Make changes
5) Test
6) Go back to #2 and repeat

When you're working with a Big Ball of Mud which doesn't readily accept testing, you need to
abbreviate a little bit. This is what I did:

1) Cleanups and small-scale refactors to simplify
2) Abstract external resources
3) Test
4) Refactor
5) Test
6) Go back to #4 and continue until you're ready to start changing functionality.

This isn't an ideal work flow, but then again the world of software is rarely an ideal place.
Sometimes you need to do a little bit of work without the safety of a test harness to save you,
because the place your at just doesn't have room for a test harness. In these cases your first
action should be to get into a testable situation and then continue with a more normal and mature
workflow.
