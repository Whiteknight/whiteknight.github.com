---
layout: post
categories: [Design]
title: Fluent Builder Interface Segregation
---

Following my previous post about [Interface Overuse](http://whiteknight.github.io/2021/05/28/everythinganinterface.html), I figured I would expand on the topic of interfaces a little bit more. As an architect I find that a lot of what I have to do is design code and systems to be consumed by developers who are often a lot less experienced than I am. It's the nature of the game, that a lot of the people who get into development eventually get out of it again. Some of them go into management, while others transition into Ops, QA, DBA, project management, product management, etc. There are a lot of off-ramps for folks with a lot of years on their resumes, and the majority of developers on many teams are on the low- to middle- portion of the experience scale. I can develop a lot of libraries and APIs for teams to use, but if they are too complicated for the target audience to grasp quickly and easily, they will be adopted slowly or not adopted at all. 

When a developer is faced with the choice to either just solve a problem right now, or try to figure out some architecture astronaut's code masturbation fantasy, the choice is often a simple one. Code is a means to an end, and code which isn't being used is pure waste. It's for this reason that many of the libraries I write for my employer, along with the libraries I write for personal means like ParserObjects, StoneFruit and CastIron, have had an inordinate amount of time spent on interface design and usage patterns. A library has to be easy to use, or people aren't going to use it.

**Interface Segregation** is a good thing. It's part of the SOLID principles after all. It is one of the most important tools in my arsenal for making clean developer interfaces. By separating the contracts from the objects which implement them, I have two axes of freedom to play with to help make the best developer experience possible. If I have a single object with a large amount of functionality, I can create multiple interfaces to cover it, so each downstream consumer of that functionality gets what looks like simple little objects instead of one big complicated one. You can even create the illusion of sharing state data between two separate objects, if they're secretly a single object behind two interfaces. You can also reuse a single object in two separate contexts, without compromising the design of either of them.

I'll give a concrete example. In my [StoneFruit project](https://github.com/Whiteknight/StoneFruit) I have a **Fluent Builder** system for creating and configuring the Engine. My Builder objects basically live two lives. In the first, the user calls methods to configure various settings and inject various dependencies. In the second life, the builder has a `.Build()` method which is called internally to create the configured objects. While it's not going to cause a serious problem, there's no reason to expose this `.Build()` method to the user. It won't do anything they want, and will only serve to create confusion when they are searching through the list of Intellisense options during Engine construction. Likewise, all the fluent configuration methods aren't used internally by the engine, so there's no particular reason why they need to be exposed there.

I'll highlight one particular class here, because there are several and they all have the same problem. `HandlerSetup` is used to setup handlers in the system. This class implements `IHandlerSetup` which exposes all the fluent configuration methods to the user, and `ISetupBuildable<IHandlers>` which exposes the `.Build()` method to the build system.

```csharp
public interface IHandlerSetup
{
    ...
}

public interface ISetupBuildable<T>
{
    T Build();
}

public class HandlerSetup : IHandlerSetup, ISetupBuildable<IHandlers>
{
    ...
}
```

Finally, I have a class `EngineBuilder` which takes all these fluent builder objects and provides access to them so the user can setup each in turn. But here's the question: What does this class look like?

In my `EngineBuilder` I can have a constructor and a few methods like this:

```csharp
public EngineBuilder(IHandlerSetup handlers = null)
{
    _handlers = handlers ?? new HandlerSetup()
}

public EngineBuilder SetupHandlers(Action<IHandlerSetup> setup)
{
    setup(_handlers);
    return this;
}

public Engine BuildEngine()
{
    var handlers = (_handlers as ISetupBuildable<IHandlers>)?.Build();
    ...
    return engine;
}
```

That's straight-forward, because if the user wants to pass in their own implementation they can do so, or if they omit it the system will provide a default. It's easy and it's safe. *But wait, there's a problem*. What if the user provides a custom implementation of `IHandlerSetup` without implementing `ISetupBuildable<IHandlers>`? Then the type cast in the `BuildEngine()` method will silently fail and the system will start up without any of the configured handlers. Massive fail. I could throw an exception in `.BuildEngine()` if the cast fails, but that seems like overkill. `EngineBuilder` asked for `IHandlerSetup`, not `ISetupBuildable<IHandlers>`, so it's really bad for usability to get exactly what you ask for and still throw an exception because it isn't what you need. 

Interface segregation seems to fail here because I have one object that not only *might implement two interfaces*, it's actually *expected to implement both* even though the compiler doesn't enforce it. But there's no way for me to annotate in the `EngineBuilder` constructor that the object needs to have two types. I mean, I guess I could do something like `EngineBuilder<THandlers> where THandlers : IHandlerSetup, ISetupBuildable<T>` but keep in mind that I have several objects like this in my system and I'm only focusing on one for simplicity. If I did this for every object that needed it, the type definition for `EngineBuilder` would be huge and messy. Plus, it would be a complexity nightmare for the user, who probably hasn't read any documentation and is only using Intellisense to try and figure out how to proceed. All not to mention that this thing might be constructed by a DI container that certainly won't be able to figure out all these generic type constraints.

Second attempt. Let's create Yet Another Interface:

```csharp
public interface IHandlerSetupBuildable : IHandlerSetup, ISetupBuildable<IHandlers>
{
}
```

I update `HandlerSetup` to implement this new interface instead of the previous two, and update `EngineBuilder`:

```csharp
public EngineBuilder(IHandlerSetupBuildable handlers = null)
{
    _handlers = handlers ?? new HandlerSetup()
}

public EngineBuilder SetupHandlers(Action<IHandlerSetup> setup)
{
    setup(_handlers);
    return this;
}

public Engine BuildEngine()
{
    var handlers = _handlers.Build();
    ...
    return engine;
}
```

Some people take issue with empty interfaces like this, but I generally think they're fine in the right situations. Notice that `BuildEngine` simplifies because we don't need the typecast and we don't need the little `?` to appease the nullability checker for cases when the typecast fails. This definitely feels better, but there are still problems. It defeats the whole purpose of interface segregation if the user is back to having one big stupid interface which represents all functionality. We want smaller interfaces for individual purposes, not a single big interface for all purposes. Also, and this is a much smaller issue, all the `IHandlerSetup` methods are exposed in `EngineBuilder`, leading future me or some other contributor to maybe forget that we're not supposed to use those methods in this particular context. Let's try again.

Third attempt. Following the **Dependency Injection** principle, `EngineBuilder` really needs to just ask for the thing it wants. In this case, EngineBuilder wants `ISetupBuildable<IHandlers>`. So let's just ask for for that.

```csharp
public EngineBuilder(ISetupBuildable<IHandlers> handlers = null)
{
    _handlers = handlers ?? new HandlerSetup()
}

public EngineBuilder SetupHandlers(Action<IHandlerSetup> setup)
{
    ...
}

public Engine BuildEngine()
{
    var handlers = _handlers.Build();
    ...
    return engine;
}
```

Great, now we still have the simple typecast-free logic in the `.BuildEngine()` method. But now we have to figure out `.SetupHandlers()`. After all the user who is configuring the system expects to be given a `IHandlerSetup` object with lots of friendly fluent methods on it. But because we're only asking for `ISetupBuildable<IHandlers>`, it's entirely possible that this is the only interface which the user implements. What if the user doesn't implement all those friendly fluent builder methods?

This is the part where I take a little step back. If the user doesn't implement an interface with fluent builder methods, maybe that's what they want? I need to trust the users when they say what they want and what they don't want. The `ISetupBuilder<IHandlers>.Build()` method is required by the system, but the fluent configuration methods are only required by a user who actually wants to use them for custom configuration. If the user's custom setup behavior is all implemented entirely inside the `.Build()` method, they don't need and won't want fluent configuration methods at all. If this is what they want, I should let them do that. With that insight now I can implement the remaining `.SetupHandlers()` method like this:

```csharp
public void SetupHandlers(Action<IHandlerSetup> setup)
{
    if (_handlers is not IHandlerSetup handlerSetup)
        throw new Exception("The provided Handler Setup object does not implement IHandlerSetup and cannot be configured");
    setup(handlerSetup);
}
```

I think this is the best design of the three options. `EngineBuilder` asks for exactly the object it wants, I don't expose any methods on `HandlerSetup` in places where those methods aren't wanted or needed, I better support the possible needs of the user, I have even put more thought into what those needs are, and if the user attempts an invalid operation I can give them a helpful message about what they did wrong at exactly the point where the mistake was made. There's the added benefit that these setup methods are called very early in the application lifecycle, so a tester will see this exception extremely quickly if it's going to happen. Putting it all together, the user can now do something like this:

```csharp
var engine = new EngineBuilder(myHandlerSetup)
    .BuildEngine();
```

or they can use the fluent interface to build it:

```csharp
var engine = new EngineBuilder()
    .SetupHandlers(handlers => ...)
    .BuildEngine();
```

Lots of flexibility and at every step Intellisense is going to tell you what your valid options are without showing you any methods you shouldn't be using. Assuming all your documentation is in place, that is.
