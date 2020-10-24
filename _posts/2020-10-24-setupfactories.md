---
layout: post
categories: [Design]
title: Refactoring to Factory Methods Example
---

I ran into an interesting little challenge during a refactoring session on [StoneFruit](https://github.com/Whiteknight/StoneFruit/) the other day. There are three objects of interest:

* The `IHandlerSource` which holds handlers and looks them up when a verb is entered
* The `HandlerSetup` which is a Builder-esque object with a fluent interface to build up a list of `IHandlerSource`s
* The `IVerbExtractor` which converts a class name into a verb. For example, `MyTestHandler` might become `"my-test"` or `"mytest"` depending on which extractor is used

During setup we create a `HandlerSetup` object, which can be used to register an `IVerbExtractor` and also a list of `IHandlerSource`s. When you type a command into a StoneFruit application, StoneFruit will parse out the verb, and use that verb to find a matching handler in the list of sources. During setup a common case, especially in the test suite, looks like this:

```csharp
handlerSetup
    .UseVerbExtractor(new ToLowerNameVerbExtractor())
    .UsePublicMethodsAsHandlers(new MyObject1());
```

The method `.UsePublicMethodsAsHandlers()` will use the public methods of the given object as handlers. So when I type the verb `mytest` on the commandline, it will be dispatched to `MyObject1.MyTest()`. Internally, that method creates an instance of `InstanceMethodHandlerSource` to hold the object instance and perform the necessary lookups. Here is a cleaned up, abbreviated version of that code:

```csharp
public IHandlerSetup UsePublicMethodsAsHandlers(object instance)
{
    return AddSource(new InstanceMethodHandlerSource(instance, _verbExtractor));
}
```

The problem, as the eagle-eyed readers might have already seen comes if the user calls the setup methods in a different order.

```csharp
handlerSetup
    .UsePublicMethodsAsHandlers(new MyObject1())
    .UseVerbExtractor(new ToLowerNameVerbExtractor());
```

Now when we go to create the `InstanceMethodHandlerSource`, we're using the default verb extractor instead of the one the user has configured. (it's a minor headache, because you have to type 'my test' instead of 'mytest', which isn't much harder but it won't be what the user expects and it won't match what's in your documentation).

The question is, how do I fix it? I considered and even tested a few possibilities before settling on a solution I like. I felt like the process I went through was instructive and the options I considered were worth discussing, so now there's this blog post about it.

## Option 1: Interface Segregation

One technique that I have used extensively in some projects, such as [Acquaintance](https://github.com/Whiteknight/Acquaintance/), is to use interface segregation on the `IHandlerSetup` object to help guide the user along and make sure things got called in order:

```csharp
public interface IHandlerSetup
{
    IHandlerSetup2 UseVerbExtractor(IVerbExtractor verbExtractor);
}

public interface IHandlerSetup2
{
    IHandlerSetup2 UsePublicMethodsAsHandlers(object instance);
}
```

In this case you can see that the compiler will only allow me to call `.UseVerbExtractor()` first, and `.UsePublicMethodsAsHandlers()` second. As an added bonus, the Intellisense and code-complete doodads in Visual Studio will help guide the user along, suggesting what to call next. Like I said, I have used this technique a lot in Acquaintance where things like a subscription might have a number of settings to fiddle with, some of which are required and some are optional. However, this technique is a lot of work because you have to create all the different interfaces (suffice it to say that there are several additional configuration methods I'm not showing for this example), and carefully layout the workflow. There are also moments of confusion for the user when she says "I'm trying to call method X but it's only giving me the option to call method Y" because she's not at the right stage of the workflow yet. So, you need lots of documentation too.

A reason why I didn't go with this approach is that `.UseVerbExtractor()`, while it likes to be called first, is completely optional. The system has a very usable default value in case you don't want to set your own, so most use-cases won't call this method. Forcing it always be called even when not needed is a waste, and setting up the segregated interfaces to make sure it is called first but can be optional is even more of a waste.

## Option 2: An Initialization Method

What if `IHandlerSource` creation was a two-step process? I could instantiate the `InstanceMethodHandlerSource` at the normal time without passing the `IVerbExtractor` instance to it, and I can call a separate `.Initialize(IVerbExtractor verbExtractor)` method later during buildup. This solution definitely would work, and is relatively easy to implement, but it comes with a few drawbacks:

1. The `IVerbExtractor` is necessary for proper functioning of the `IHandlerSource`, so if we separate construction from initialization, there's a period of time where the object is in an invalid, unusable state. 
2. The `IVerbExtractor` field, and any other internal state which derives from it, cannot be marked `readonly` so we're adding more mutable state to the system
3. The `.Initialize()` method either expands the size of the `IHandlerSource` interface or else it necessitates the creation and consistent use of a second `IHandlerSourceInitializable` interface. 
4. We would need to put safety checks in place to make sure that the `.Initialize()` method isn't called a second time on accident

I actually put most of the code in place for this option, but I was so unhappy with the results that I bailed out and started looking for a third option.

## Option 3: Factory Methods

Let's go back to our original implementation of `.UsePublicMethodsAsHandlers()`:

```csharp
public IHandlerSetup UsePublicMethodsAsHandlers(object instance)
{
    return AddSource(new InstanceMethodHandlerSource(instance, _verbExtractor));
}
```

What if, instead of creating the instance immediately, we instead register a factory method to create it later:

```csharp
public IHandlerSetup UsePublicMethodsAsHandlers(object instance)
{
    return AddSource(verbExtractor => new InstanceMethodHandlerSource(instance, verbExtractor));
}
```

Now we can call `IHandlerSetup` methods in any order, and we know that we won't have to access `verbExtractor` until buildup time. There's no complicated interface redesigns and no sloppy `.Initialize()` methods polluting the code. 

## Review

The `.Initialize()` method approach, which I see far too often in the wild, is a really lousy solution and I think most developers should cut that technique out of their toolbox. It pollutes the interface of an object, it increases mutable state, and it creates situations where an object must be in an invalid state (between constructor and initializer). I shouldn't have even considered it in this case, honestly.

The segregated interfaces approach is a very powerful tool but it requires a lot of effort. I would recommend this only in cases where fluent interfaces are particularly complex and you want to make sure downstream users have the full assistance of the IDE to help guide them through the process.

The factory method approach is definitely the right way to go in this case because it keeps all the interfaces clean, makes sure objects are always consistent, and gives better control of exactly when we create objects. There are some performance implications of this approach, because the anonymous delegate requires a few memory allocations, but in my case because it's only happening once at startup time I didn't think it would be a problem.

I thought it was a little bit instructive to show not only which approaches I tried, but why I considered them and ultimately why I used one instead of the others.