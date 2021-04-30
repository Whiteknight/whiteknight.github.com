---
layout: post
categories: [Design]
title: The Singleton
---

When you ask a person what design patterns they know about and use, the **Singleton** is almost always one of the first ones mentioned. I don't know if this is because lots of people are still using it, or if it's just so much more simple of a concept than most other design patterns. Maybe it's a combination of both. While it may be common there are several problems which have lead some corners to really hate on it. Today I'm going to talk a little bit about our friend the Singleton, how to use it, and when to avoid it.

## Definitions

First, let's look at some definitions of the Singleton pattern from a variety of sources:

* [The singleton pattern is a design pattern that restricts the instantiation of a class to one object](https://www.geeksforgeeks.org/singleton-design-pattern/)
* [Singleton is a creational design pattern that lets you ensure that a class has only one instance, while providing a global access point to this instance](https://refactoring.guru/design-patterns/singleton)
* [Application needs one, and only one, instance of an object. Additionally, lazy initialization and global access are necessary](https://sourcemaking.com/design_patterns/singleton)
* [This pattern involves a single class which is responsible to create an object while making sure that only single object gets created](https://www.tutorialspoint.com/design_pattern/singleton_pattern.htm)
* [Ensure a class only has one instance, and provide a global point of access to it](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612) 

The last definition comes directly from the famous **Design Patterns** book by the Gang of Four, which is as close to a definitive source as anything, and which adds this additional stipulation:

> ...the sole instance should be extensible by subclassing, and clients should be able to use an extended instance without modifying their code.

We can already see some differences between the various definitions of the pattern. Some definitions restrict the number of instances you are allowed to *create*, while others restrict the number you are allowed to *have*. This is a bit of a pedantic difference, admittedly, but can affect implementation. Some definitions require lazy initialization. Some definitions require a global point of access to the single instance. The first website uses simple static initialization, the third website uses double-check locking, and the fourth website uses single-check initialization.

## Creating One Instance

A key difference between some of these descriptions and the example implementations the sites provide, is whether exactly one instance is ever created, or whether exactly one instance is used by the application. These are related but separate ideas. First, let's look at some implementations:

### Simple Static Initialization

```csharp
public class MySingleton
{
    private readonly static MySingleton _instance = new MySingleton();

    public static MySingleton Instance() => _instance;

    private MySingleton() { }
}
```

The use of the private constructor means that no other class can create an instance of the singleton without using reflection (and if you're willing to violate access rules via reflection, the game is already lost). Here we create the singleton instance on static initialization of the class and return a reference to that single object on demand. A big problem here, at least for C# and I think a few other languages as well, is that there is no hard guarantee that the static constructor for the class is executed exactly once. The runtime executes the static constructor on the first time the class is referenced from the code, but it isn't behind a lock. If you are referencing it simultaneously from multiple threads, it's possible that you create multiple instances of the singleton, but only save the last one to the `_instance` property. If the problem is that the initialization logic is expensive in terms of time or resources, creating multiple instances and then discarding all but one of them doesn't help. Likewise if your instance needs to gain exclusive access to a global resource, creating multiple instances and then possibly throwing away the one instance which has successfully obtained that resource will also be a problem. 

(Notice also that using `Lazy<MySingleton>` instead doesn't completely solve the problem though it will help. It's still entirely possible that separate threads create separate instances of the `Lazy<MySingleton>` and then also instantiate the singleton separately in those two instances. Using Lazy helps with the desire not to instantiate the singleton instance until it is actually needed, but doesn't solve the threading problem).

Using this solution, at least in C#, doesn't guarantee that a single instance is created and doesn't guarantee that all threads access the same instance (at least on the first, conflicted access. Subsequent accesses should all see the "final" version of it.)

### Single-Check Initialization

```csharp
public class MySingleton
{
    private volatile static MySingleton _instance;

    public static MySingleton Instance()
    {
        if (_instance == null)
            _instance = new MySingleton();
        return _instance;
    }

    private MySingleton() { }
}
```

This solution helps with the threading issue but isn't perfect. There's still ample opportunity (especially if the constructor for `MySingleton` is long-running) for thread preemption after the null check but before the instance is assigned to the property. The `volatile` keyword on the property is optional here and it doesn't solve the problem but can help in some cases.

### Double-Check Locking

```csharp
public class MySingleton
{
    private static object _lockObject = new object();
    private static MySingleton _instance;

    public static MySingleton Instance()
    {
        if (_instance == null)
        {
            lock (_lockObject)
            {
                if (instance == null)
                    _instance = new MySingleton();
            }
        }
            
        return _instance;
    }

    private MySingleton() { }
}
```

This implementation is generally considered the "best" for this specific idea, but I think there's probably still a very small opportunity for preemption to happen while setting the `_lockObject` and thus the double-check lock could lock two separate objects and then both threads proceed to create separate instances of the singleton. This risk probably goes away if we mark `_lockObject` as `volatile`, but I haven't verified that. In either case, this is a very good solution to the problem, and the one that people should generally use.

### Having a Single Instance

In many use cases it may not strictly matter if you create multiple instances, only that you use a single instance through the duration of the application. In these cases you can get a much better guarantee of thread-safety using atomic functions:

```csharp
public class MySingleton
{
    private static MySingleton _instance;

    public static MySingleton Instance()
    {
        var thisInstance = Interlocked.CompareExchange(ref instance, null, null);
        if (thisInstance != null)
            return thisInstance;
        
        thisInstance = new MySingleton();
        var originalInstance = Interlocked.CompareExchange(ref _instance, thisInstance, null);
        return originalInstance ?? thisInstance;
    }

    private MySingleton() { }
}
```

In this implementation it is still entirely possible that the thread preempts between the first access of the property and the time that the property is set. This means that it's possible to call the constructor more than once. However the `Interlocked.CompareExchange` forces the property to only be set to a new instance once. All downstream consumers of `Instance()` will always get the same instance of the object, all without locks. Again, you still might create multiple instances but this implementation guarantees that downstream code will only use one.

There's probably an implementation to be had that's belt-and-suspenders safe, but using `CompareExchange` to allocate a `_lockObject` only once and then using double-check locking on the `_lockObject` to get your single instance, but at some point we've dramatically over-engineered a simple access problem.

## Subclassing And Extending

While most web sources don't mention it, the Design Patterns book requires that the singleton should be able to be subclassed or extended in some way. None of the code examples I showed above directly supports this, though they can be modified to do so. The book offers two options:

1. Instances can exist in separate libraries, and libraries be selected at link-time depending on which implementation you want to use
2. The instantiation routine can access some external source of information, such as a configuration file or environmental variable, to select which class to use.

Both of these are kind of tricky, in the first case you have to do things at link time which can be a big headache working in most modern development environments (and it's not so great to do at the C/C++ level with `Makefile` either). You're going to be juggling multiple builds, and if you have more than one singleton in your system you are going to be juggling an exponential number of combinations. In the second case you have more flexibility to make changes, but you will need to add additional validation and instantiation logic to build the type by reflection and verify that the instance is the correct type.

In neither of these cases do you have a lot of good flexibility to compose a solution, such as through the use of the Decorator pattern or some other kind of wrapper/adaptor. At least, not without adding even more logic to figure out dependencies.

## On `Lazy<T>`

I mentioned briefly above, `Lazy<T>` helps with lazy initialization in many cases but doesn't really have a good role to play in any of these. The `Instance()` method examples I showed above are already doing lazy initialization, so adding a reference to `Lazy<T>` just adds a level of indirection without adding much benefit to the system.

## Problems with Singleton

All the examples I showed above suffer from the same crippling problem: Because there is a single, static, global instance of the singleton, you don't have easy control over what instance you want to get. This means that classes become more *coupled*. Every class which needs to access the singleton is coupled directly to the `MySingleton` class. Likewise, in your unit test suite, you don't have any opportunity to inject a separate instance per test (you can, as I mentioned in the discussion on Subclassing above, use a config value to choose a single instance for your entire test suite, but you can't select them per-test if your tests all run together in a single process, like most do).  If your singleton needs to have different data in different tests, or (even worse) if it needs to mutate data within your tests, you're going to have a real bad time.

There are other cases where you may want to use a different implementation in a context-specific way. For example, in a Web Service situation you may want to set up a "singleton" per request context, not per application lifetime. Maybe you want to use a separate `SecurityPolicy` singleton for authenticated admins than you do for other users, for example. Or maybe you have a `DataAccess` singleton which provides database access, but then somebody from architecture wants to switch to use a third-party CRM system to hold user profile data, so now that vertical slice wants to use a different data access singleton for that data, but the original implementation for everything else.

For these reasons the use of singleton via public static accessors like this is generally frowned upon by many people. I personally would discourage these kinds of implementations of the patter, for exactly these reasons. All those code samples I showed above? **I don't want you to use them**.

## With a Container

So if all these static accessors aren't the solution, even the fancy Double-Check Locking one, what is? Enter the **Dependency Injection Container**.

```csharp
services.AddSingleton<IMySingleton>(() => MySingleton.Instance());
```

A solution like this is the best of (almost) all worlds. The container is probably being set up in your **Composite Root**, which executes at program start up, in a place of your choosing, before your threads get launched. You can completely eliminate the possibility of concurrency issues by setting this up in a place where there is no concurrency. The static method can still jump through hoops to make sure only a single instance is created per application in case somebody else couples directly to that method elsewhere in the application (this is exactly why we have code reviews, to catch these kinds of errors! But if you don't catch it, at least you aren't violating some kind of invariant or tanking performance). Using the `IMySingleton` interface gives you an abstraction behind which you can provide one of many implementations, including subclasses, wrappers, decorators, composites, etc. You can inject this value using constructor injection throughout your object graph, which means you can easily provide new instances in separate vertical slices, separate execution contexts, and in individual unit tests. This implementation is actually supported by the Design Patterns book, where the authors write:

> A more flexible approach uses a **registry of singletons**. Instead of having `Instance` define the set of possible Singleton classes, the Singleton classes can register their singleton instance by name in a well-known repository.

Service Locator and DI/IoC containers weren't talked about in the Design Patterns book, but it should be pretty clear that both of those represent obvious ways to implement a "registry of services"

If you aren't using a DI container, you can still manage a single instance using constructor injection throughout your application. Just create the single instance at the beginning of your application and pass it through constructors when you create the rest of your execution pipeline.

There are two downfalls to this style. First, The instance is only singleton *per container*, not per application. If your container creates multiple subcontainers for some reason, each one may have a separate instance of the singleton (check documentation for your container). This might work in your use case, but it might not, in which case you need to do a little extra work to make sure the same instance is used by all containers. Second, the instantiation might not be created lazily, depending how your container works or how you set up the registration. 

## My Definition

In my mind, singleton is about there being a single instance of a particular type for your particular context. Your context may be the entire application, or by module, or by user request, depending on your needs. The Singleton pattern doesn't strictly require either lazy initialization or single-instantiation, though these might be additional requirements for you. The important part is that you only work with a single instance in your context, and whatever mechanism you use to enforce the singleness rule within that context should be sufficient. If you want to enforce singleness using double-check locking in a static method, do that (but don't be surprised when unit-testing becomes tougher). I know that I personally would probably never use Singletons without a DI container to manage it, but that's a personal style issue more than a hard technical requirement.

