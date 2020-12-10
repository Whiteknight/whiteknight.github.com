---
layout: post
categories: [Patterns]
title: Dynamic Visitor Pattern
---

The [ParserObjects](https://github.com/Whiteknight/ParserObjects) library builds up parsers as graphs of smaller parsers and parser combinators. When talking about operations across graphs, it's very common to talk about the **Visitor pattern** as a way to traverse the graph without having to hard-code traversal rules for every structure inside the graph (replace "graph" with "list" or "tree" if your use-case is simpler than mine). Today I'm going to talk about the Visitor pattern in general, and then the technique used in ParserObjects to implement the Visitor pattern. Finally, I'll go over one (small) issue with the implementation from ParserObjects that caught me off guard.

## The Setup

Here's a quick eCommerce example: I have an `Order`, the `Order` has `LineItems`, and each `LineItem` has a `UnitPrice` and a `Quantity`. If I want to get the total price of the entire order, I can do something like:

```csharp
var totalPrice = order.LineItems.Select(li => li.UnitPrice * li.Quantity).Sum();
```

That's a pretty easy example, and if the world were a simple place we could end the discussion there. *But it ain't*. Now the business folks come in and they want to include shipping and shipping options, and those all might have cost too. So now our modest calculation turns into something like this:

```csharp
var totalPrice = 
    order.LineItems.Select(li => li.UnitPrice * li.Quantity).Sum()
    + order.Shipping.Price
    + order.Shipping.Options.Select(o => Price).Sum();
```

Fine. Great. Whatever. We have it and it works and the world is still relatively simple. Until...

Now the business folks say our products need to be *configurable*. Items can be customized and personalized, and different options come with all sorts of different costs. 

```csharp
var totalPrice = order.LineItems
    .Select(li => 
        (li.UnitPrice * li.Quantity)
        + li.Options.Select(o => p.Price).Sum()
    ).Sum()
    + order.Shipping.Price
    + order.Shipping.Options.Select(o => Price).Sum();
```

Surely this must be the end of the madness! Nope. Now the business folks remind you that we need to calculate **tax** (at least, on taxable items), and tax changes depending on location. And then somebody in the back of the room reminds everybody that this is a lot like *discounts* and *coupons*, which only apply to some items but not others, but they aren't going to ask for that until next sprint so don't worry about it today. Then another business person pipes up and says that really line items should be part of shipments, because the order might ship in multiple shipments, from multiple separate warehouses. Now our calculation is getting out of hand, and every time we add a new feature, we have to change the object graph, and then we also need to change the calculation which requires knowledge about the object graph.  Luckily, there's a way to separate the form from the function.

## The Visitor Pattern

The Visitor pattern is helpful here because it allows us to separate out the implementation of graph traversal, from the operations which get applied to those traversals. In the case above, we are going over every item in the `Order` graph and summing up prices, but there are a lot of other operations we could apply to a graph as well: find and/or replace a particular object in the graph, Serialize the object graph to a string or even a list of objects, look for particular features in the graph, etc. Plus we may want to do all these operations on sub-graphs instead of the entire graph.

The basic visitor pattern uses a double-dispatch mechanism. The visitor calls `.Visit()` on a visitable object without knowing it's specific type information. The object then calls back to an `.Accept` method, which includes the specific type information. Each object also needs to tell the visitor where to find the next objects in the graph (the children). Here's a very quick example of the above using static typing, with some redundant or uninteresting parts omitted:

```csharp
public interface IVisitable
{
    void Visit(IMyVisitor visitor);
}

public interface IMyVisitor
{
    void Visit(IVisitable);
    void Accept(Order order);
    void Accept(LineItem lineItem);
    void Accept(LineItemOption option);
    void Accept(Shipping shipping);
    void Accept(ShippingOption option);
}

public class Order : IVisitable
{
    ...
    public void Visit(IMyVisitor visitor)
    {
        visitor.Accept(this);
        foreach (var li in LineItems)
            visitor.Visit(li);
        visitor.Visit(Shipping);
    }
}

public class LineItem : IVisitable
{
    ...
    public void Visit(IMyVisitor visitor)
    {
        visitor.Accept(this);
        foreach (var lio in LineItemOptions)
            visitor.Visit(lio);
    }
}

public class PriceCalculatingVisitor : IMyVisitor
{
    public static decimal CalculatePrice(IVisitable visitable)

    public decimal Price { get; private set; }
    private decimal TaxRate { get; set; }
    
    public void Visit(IVisitable visitable) => visitable?.Visit(this);

    public void Accept(Order order)
    {
        TaxRate = order.GetTaxRate();
    }

    public void Accept(LineItem li)
    {
        var price = li.Quantity * li.UnitPrice;
        if (li.IsTaxable)
            price = price * (1.0 + TaxRate);
        Price += price;
    }

    ...
}

```

Sorry for the long example, but this is not a concise pattern. The double-dispatch, where the visitor calls `.Visit` and the visited calls `.Accept` is because the visitor doesn't have the necessary type information in the `.Visit` method. The visited object knows it's own type, so when it calls `.Accept`, that type information is used to dispatch to the appropriate override. The up-sides to this pattern are clear:

1. The `PriceCalculatingVisitor` doesn't have to know about the structure of the graph. It doesn't care if `LineItems` live on the `Order` or if they get moved into shipments, or anywhere else.
2. The various model objects (`Order`, `LineItem`, `Shipping`, etc) don't have to know anything about the ever-changing price calculation, or whatever other calculations or operations other visitors might perform.

But there are problems here as well as benefits. The `IMyVisitor` interface has to have `.Accept()` method overrides for every single visitable type, and they all have to be `public`. Plus, every single `IMyVisitor` implementation must provide those methods, even for types that don't need any attention at all. For example, a visitor which is only trying to get information about products for inventory purposes will still have to provide empty `.Accept` methods for shipping objects. For small and uncomplicated object graphs these might not be problems, but for complicated situations it becomes bloated and ugly. Worst of all it becomes, dare I say it, *inelegant*!

In the next sprint the business guys come in and tell you that your customers would like to provide some of their own custom options classes to the system, which need prices to be calculated in a special way. These custom options from your customers will depend on whether or not *their customers* are members of *your customer's* member rewards program. Now we need the visitor to suddenly be able to support types from the downstream user, and factor in data that your system cannot possibly have access to.

## Dynamic Visitors

This is the situation ParserObjects is in. ParserObjects provides a large suite of built-in parser types, but it also publically exposes the `IParser` interface. Downstream users can provide their own implementations, and those are expected to *just work* with all built-in visitor types. How do we do that? Luckily for us, C# provides the `dynamic` keyword, which gives us exactly what we need. Here's the same Visitor example, implemented with `dynamic` instead:

```csharp
public interface IVisitable
{
    IEnumerable<IVisitable> GetChildren();
}

public class PriceCalculatingVisitor
{
    public decimal Price { get; private set; }

    public sealed void Visit(IVisitable visitable)
    {
        ((dynamic)this).Accept((dynamic)visitable)
        var children = visitable.GetChildren();
        foreach (var child in children)
            Visit(child);
    }

    private void Accept(IVisitable visitiable)
    {
        // default implementation, called only when a better override isn't available
    }

    protected virtual void Accept(Order order)
    {
        ...
    }

    protected virtual void Accept(LineItem li)
    {
        ...
    }
}

public static class VisitableExtensions
{
    public static decimal CalculateTotalPrice(this IVisitable visitable)
    {
        var visitor = new PriceCalculatingVisitor();
        visitor.Visit(visitable);
        return visitor.Price;
    }
}
```

And that's it. We don't need the double-dispatch where the visitor calls `.Visit` and the visited calls `.Accept`, the `dynamic` keyword makes sure that the `visitable` object's type is used for dispatch logic at runtime, not compile time, so we can get the type information without having to call out and call back in. The `((dynamic)this).Accept()` part means that the correct override for `.Accept` is picked at runtime, based on the runtime type of the `visitable`, not the compile-time `IVisitable` value. Notice also how the `.Accept` methods are marked `protected virtual`. This is a hint that this `PriceCalculatingVisitor` can be inherited from down-stream customer code, and new `.Accept` method variants can be added there as well. Customers can override our price-calculating logic, or they can add new `.Accept` method overrides to cover their own custom types, and it *all just works*. Plus, the visitor only needs to define `.Accept` overrides for the types it actually cares about. This is great, except there's one small hangup.

## Variants

There are a few other things you could do in your Visitor pattern to meet different requirements:

1. You could keep a list of objects that were already visited so that cycles in your object graph do not lead to infinite recursion. This isn't a concern if your graph contains no cycles.
2. You could pass a State object to all your accept methods, if you needed to keep track of more detailed state information than just a single `Price` property.
3. You could use some kind of a **Trampoline** loop instead of recursion, so that a deep object graph doesn't create problems with recursion or the stack.

One thing that I would generally recommend against with visitors is to avoid over-abstraction. You don't need, and probably don't want, a single `VisitorBase` object which does your visiting and dispatch and state management, and then inherit from that for all your particular visitor implementations. The logic that you need can change so much from visitor to visitor that it's almost better to just write each one from scratch. A searching visitor may want to bail out immediately when an object is found, but many other visitors won't. Also, in my experience, there aren't generally a lot of good opportunities to parallelize visiting operations, although there are some. Trying to build all these possibilities and alternatives into a visitor base class, when most implementations will need to opt-out of them, is wasted effort.

## Unit Testing

The dynamic approach seems pretty great, but because there's no double-dispatch there's no good compile-time way to know that every `IVisitable` implementation has a corresponding `.Accept` method override inside the visitor. The beauty of this approach is that you can omit certain `.Accept` overrides if you don't need them, but in some cases you want to make sure that your visitor does have them all. The only real way to find out would be at runtime, when you call `.Visit` on an object, and end up in the default `.Accept` method instead. We would like to have a unit test in our test suite that tells us whether we're missing any overrides, if all overrides are required.

Using reflection, we can put together a basic unit test like this:

```csharp
[Test]
public void AllAcceptOverridesExist()
{
    // first get all visitables
    var visitableTypes = typeof(IVisitable).Assembly
        .GetTypes()
        .Where(t => t.IsPublic && !t.IsAbstract && typeof(IVisitable).IsAssignableFrom(t))
        .ToList();

    // now get all .Accept variants, and get the first parameter type from each
    var acceptMethodTypes = typeof(PriceCalculatingVisitor)
        .GetMethods(BindingFlags.Instance | BindingFlags.NonPublic)
        .Where(m => m.Name == "Accept")
        .Select(m => m.GetParameters()[0])
        .ToHashSet();

    foreach (var type in visitableTypes)
    {
        if (!acceptMethodTypes.Contains(type))
            Assert.Fail("Missing an Accept variant!");
    }
}
```

Test looks straight forward except for one little problem: It fails for generic types. This, in fact, is the reason why I'm writing this post today, because I was bitten by this very issue. Our down-stream customer has some types which are generic:

```csharp
public class CustomerPriceCalculatingVisitor : PriceCalculatingVisitor
{
    protected void Accept<TData>(CustomLineItemOption<TData> option)
    {
        ...
    }
}
```

This method works correctly at runtime, the visitor correctly dispatches the option to the correct `.Accept` override, filling in the generic type parameter according to the type of the `option` variable. However, the unit test tells you that there is no matching `.Accept` override for `CustomLineItemOption<TData>`. Inspecting the two types in the debugger shows something curious. The `type` is listed as ``{Name = CustomLineItemOption`1, FullName = CustomerLib.CustomLineItemOption`1 }`` but the equivalent entry in `acceptMethodTypes` is listed as: ``{Name = CustomLineItemOption`1, FullName = null }``. WTF? They're two different objects? `FullName` is null? The Microsoft documentation has this to say about `Type.FullName`:

> The fully qualified name of the type, including its namespace but not its assembly; **or `null`** if the current instance represents a generic type parameter, an array type, pointer type, or `byref` type based on a type parameter, or a **generic type that is not a generic type definition but contains unresolved type parameters**.

What's happening here, as best as I can understand it, is that the type `CustomLineItemOption<TData>` is not free to change however, but is instead beholden to the type parameter from `Accept<TData>()`. That is, the parameter type is not unbound. It is bound to the generic parameter type of the method, but the method is unbound. So, in this case, `method.GetParameters()[0] != typeof(CustomLineItemOption<>)` even though it looks like it should be the same. I don't know if this is done because of the dependency relationship between the parameter type and the method declaration, or the fact that they might have differing generic type constraints, or both. Maybe somebody who knows the details of this issue can let me know exactly what's going on here.

Because `.FullName` isn't filled in, and it doesn't look like the parameter type has any reference back to the original generic type definition, it doesn't seem like there's any obvious way to do an exact match in this case. The best I can come up with in my own tests is this:

```csharp
foreach (var type in visitableTypes)
{
    bool exists = acceptMethodTypes
        .Any(am => 
            am.Namespace == type.Namespace
            && am.Name == type.Name
            && am.BaseType == type.BaseType
            && am.DeclaringType == type.DeclaringType
        );
    if (!exists)
        Assert.Fail("Missing an Accept variant!");
}
```

...And I'm not certain that this is sufficient in all cases. Another approach might be to try and bind all the type parameters, and then compare the resulting constructed generic types, but this might be tricky if there are type constraints on the method or the type that need to be satisfied.

## Overview

The Visitor pattern, like many other patterns, isn't strictly rigid. The big idea is that you separate out graph traversal from operations that require graph traversal. These can be operations that search the graph, modify the graph, or generate some new representation from the information inside the graph. The double-dispatch mechanism from the first implementation is helpful to deal with hidden type information, but creates a solution that is bloated and inflexible. The `dynamic` version is lighter and more flexible. As I mentioned in the Variants section above, there are other things you can do to change the way Visitor works as well, depending on your needs. It's a cool pattern and can help with a lot of operations, but if you put too much effort into abstractions and shared code, you might end up wasting a lot of time.

The unit-test issue was obnoxious, but it's not exactly a deal-breaker. The fact that I can't exactly match an unbound generic type to a parameter type from an unbound generic method made one unit test longer than it should have been, but otherwise isn't a huge problem. Just keep an eye on it for your own test suite.
