---
layout: post
categories: [Design, Theory]
title: On Access Modifiers and Inheritance
---

A question came up on Reddit the other day about [The "sealed override" modifier](https://www.reddit.com/r/csharp/comments/dd5c1o/til_sealed_override_modifier/). I mentioned in my reply that this modifier, like several other  modifiers, [shouldn't be used](https://www.reddit.com/r/csharp/comments/dd5c1o/til_sealed_override_modifier/f2eeot2?utm_source=share&utm_medium=web2x). I caught a little bit of flak for my opinion there, but I defend what I say and decided to even write a blog post about it. Of all the things I said shouldn't be used, the response in favor of the "internal" keyword was probably the strongest.

**TL;DR**: just about everything I have to say in this blog post boils down to the idea that **concrete inheritance is pretty much a bad thing** and should be replaced with interface inheritance or delegation/composition in almost all cases. All the keywords and mechanisms which support concrete inheritance are, therefore, bad ("protected", "virtual", "abstract", "override", "base" and "sealed" which should be the default for all classes and methods if C# were a better language).

When talking about developing something for which access modifiers might be a concern, there are two cases to consider: The case where I'm producing an application which I host or distribute and users interact with some sort of user interface, and the case where I'm producing a library which downstream developers will interact with through code. In the first case, because only a limited number of people are looking at the code (my team and myself) and so long as this limited number are able to do what they need to do, they can organize the code however they see fit. It's probably easiest to just make everything public so everything is easily exposed to unit test assemblies if necessary, and all features are quickly discoverable.

It's the second case, where we're making a library for public consumption, which is by far the most interesting because the number of people interacting with the code is potentially unbounded. It's imperative that we follow some basic best-practice design guidelines to save everybody a huge amount of headache.

## What's Wrong with Inheritance?

Inheritance is one of the four pillars of Object Oriented Design and of all the pillars it's the one that typically gets the most support from OO programming language. Inheriting is intentionally made easy, C# only requires a simple colon and the name of a type to make it happen. Of inheritance, there are two concepts worth considering: inheriting interfaces and inheriting classes. Interface inheritance is cool: Do as much of it as you want, and remember to follow the Interface Segregation Principle as best as possible. Class inheritance on the other hand is awful. You can typically only inherit a single class (some languages allow multiple inheritance, but C# luckily doesn't and I hope it never does). Class inheritance has several problems:

1. It imposes structure on child classes, breaking encapsulation. Any class should be able to be implemented in the best way for its purpose, and the internal details of the class should be completely private to that class. A parent class imposing structure on a child class breaks encapsulation for both and limits freedom.
1. It hides shared code ("protected" methods, etc) so that child classes can access it but unit tests can't directly test it and downstream users can't access it without inheritance
1. It forces mutable state to be handled across multiple classes, making a bad situation even worse. 

While it's easy to find sites on the internet that talk about [composition being preferred over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance), I would take it a step further. It's not just a *preference* but instead should be a *rule*. My hypothesis is that avoiding class inheritance *in all cases* makes code better. 

(Note that some poorly-designed libraries require inheritance as part of their public interface, and sometimes your code may be forced to use it to interact with these libraries. That's a need, and you should do as little of it as required. Send a note to the maintainers of these libraries and strongly suggest an alternative.)

Class inheritance basically serves three purposes and each of these three can be better served by other mechanisms:

1. **Implementing Polymorphism** : Use an interface instead. Interfaces enforce a contract *without imposing structure* on the internal design of your class, enabling proper encapsulation. 
1. **Sharing State** : Factor your shared state out into a State, Context, or Boundary object and delegate access to that instead. This keeps all your mutable state logic in a single place, so it's easier to manage and keep consistent.
1. **Sharing Code** : Factor your shared state into into another object; for example a Service, Domain Model, or Strategy object. You can expose the public methods on these objects to rigorous unit testing and more flexible reuse.

You may just find when you're done refactoring that you have a superior system which is easier to maintain and more flexible to change. In fact, I guarantee it.

## Limiting Visibility

Let's say we want to limit visibility of a class member. After all, "information hiding" is part of encapsulation and let's assume encapsulation is generally good. Information hiding is, effectively, the only purpose of access modifiers like `private`, `protected`, `private protected` or `protected internal`. Let's consider an alternative approach:

```csharp

public interface IMyContract
{
	void DoAThing();
}

public class MyConcreteType : IMyContract
{
	public MyConcreteType(string value) 
	{
		Value = value;
	}
	
	public string Value { get; }
	
	public void DoAThing() { ... }
}
```

Notice the structure we have here, where the `IMyContract` interface exposes a single method, but the `MyConcreteType` also exposes a public property. We can only access the value of that property if the instance is type `MyConcreteType`, but it suddenly becomes invisible as soon as we cast that instance to `IMyContract`. Now think about one of the common structures used by many applications: An Aggregate Root where a DI container is set up, and the services of the application are resolved from the container to handle each new job/request. In a situation like this, we would create a `MyConcreteType` instance and then register it with the DI container as `IMyContract`. When the application service requests an `IMyContract` the DI container will provide this instance, but the `Value` property will be hidden from them. But, and this is the crucial bit, the property *isn't hidden from any piece of code which knows about the underlying type*. If you have the information about the concrete type you can interact with it in a more intimate and powerful way, and if all you have is the contract then those are the rules you must play by. The applications of this approach are numerous. You can have a bunch of hidden "Builder-like" methods to help with initialization of your object and then when it's fully-initialized you can cast it to a read-only contract type and hide all the mutator methods. Think about casting `List<T>` to `IReadOnlyList<T>` to disallow mutation by hiding the mutator methods. Or you can use Interface Segregation to make a single object appear to be multiple objects with mysteriously-shared state. Or you can hide complicated "internal-only" details from the casual downstream user by not exposing them in any of the interfaces which the downstream user is consuming. You can use interface segregation to choose which "view" of an object is visible in any given context. The possibilities are numerous, and the power and flexibility of this approach dwarf the supposed utility of  worthless access modifiers.

Here's another approach:

```csharp
public class MyClassA : IMyContract
{
	public MyClassA(MyData data)
	{
		_data = data;
	}
	
	private readonly MyData _data;
	
	public string Value => _data.Value;
	
	public void DoAThing() { ... }
}

public class MyClassB : IMyContract
{
	public MyClassB(MyData data)
	{
		_data = data;
	}
	
	private readonly MyData _data;
	
	public string Value => _data.Value;
	
	public void DoAThing() { ... }
}
```

In this case, objects of `MyClassA` and `MyClassB`, when cast to `IMyContract` sure do seem to inherit from a single concrete class and they do seem to share some data, when they actually don't share any concrete inheritance relationship at all. The "sharing" of data is done by delegating to a third object (over which we have much more control), and we only expose information from that third object from the first two as needed. Hence, we have used `MyClassA` and `MyClassB` as facades to limit visibility of information from `MyData` in some places, while we can get full access to all the data in other places. Think of this as something like a Facade, Decorator or Adaptor pattern, but instead of adding new features you provide a very limited protected view of features to provide different visibility on a single underlying construct in different contexts. Again, there's a lot of power in this approach, and it's superior in almost all ways to inheritance and access modifiers.

## The Problem with Internal

The `internal` keyword is a headache and if I could make any changes to the C# language removing it would be pretty high on my list of improvements. The problems of `internal` are compounded by the fact that it is the default access modifier for classes and so is used much more than it would be otherwise. My opinion on this matter, unpopular as it may be, is this: **all non-nested classes and structs should be public**. Why do I say this? Reasons:

1. Any class which isn't public is not able to be easily and directly unit tested during development
1.1. You could use reflection to access these parts, but that adds a lot of brittleness to your suite,
1.1. You could include unit test code in your production assembly, which is generally frowned upon, or
1.1. You could just skip unit-testing these bits entirely because who really cares? (Don't say this. You should care)
1. No abstractions are perfect. You may think hiding details from your users is a benefit, but when the abstraction inevitably leaks and the users need more flexibility, it's going to require a change to expose something which should have just been exposed in the first place
1.1. You can make clear, through class naming, namespace naming and good-old public documentation, which classes are the preferred pretty abstractions and which classes are the super-user nitty-gritty.
1. If the class has any value at all, it likely has value to downstream users as well, so it should be exposed to them
1. Giving users all the building blocks to compose their own solutions, as opposed to limiting them to just using narrow abstractions, makes your library more flexible, more usable, and more powerful.

### A Challenge

In true scientific fashion, a hypothesis must be falsifiable to be a valid hypothesis. Maybe I think the way I do because I've never seen a counter-example (I'm not experienced enough) or I can't imagine a counter-example (I'm not creative enough). To that end, I'll pose a challenge:

**I'll change my mind if anybody shows me a single example of code which uses any of these modifiers (`internal`, `private protected` or `protected internal`, `sealed` or `override`) to improve code usability either by programmers in the development team of that software or downstream developers of client applications, where the code could not be more usable by switching these modifiers to `public` and improving the usability of the component.** I won't hold my breath, I don't expect to see any real examples, just a bunch of wishy-washy "never say never" or "they probably have a valid usage somewhere, I imagine".

## The New Modifier

You can use the `new` keyword as a member modifier to basically allow you to override a method from a parent class even if it's marked `sealed` (this feature effectively makes the `sealed` keyword worthless, so I recommend at least one of the two be removed. We can either seal a class or we can't). Here's a blurb from Microsoft's own documentation:

	Although you can hide members without using the `new` modifier, you get a compiler warning. If you use `new` to explicitly hide a member, it suppresses this warning.
	
So basically it allows you to do a bad thing and also hide warnings so people don't know you're doing it. It's the perfect crime.

(I said "override" above but actually the `new` modifier does "name-hiding". Bonus points to any reader who can explain, with code examples, what is the difference between name-hiding and overriding, and different behaviors of the `new` and `override` modifiers. If, in studying for this challenge in order to prove how much of an idiot I am, you realize that these keywords really are stupid you get even more Bonus points.)

Even among people who use inheritence regularly the `new` modifier is almost never used, and people who choose not to use inheritance don't have to deal with it at all. It's just another small benefit of the approach.

## On Sealed Override

I have a class. The class can be inherited, but some of the methods cannot be. Why not? Am I so perfect that what I write can never be improved upon? I'm going to use inheritance but I'm going to force downstream developers to use delegation because "do as I say, not as I do"? If I have a class which can be inherited but a method which cannot be, what I'm effectively doing is limiting flexibility and hobbling downstream developers. If you don't want people to override parts of your algorithm just mark your whole class sealed and force people to delegate if they want to use parts of it. In fact, mark all your classes sealed and don't use class inheritance. It's a superior approach!

Don't override. Inject and delegate. I promise you your code will benefit from the change. If you're not doing any of that, you don't need the `override` keyword at all. Mark your classes sealed and keep your methods clean and unencumbered.




