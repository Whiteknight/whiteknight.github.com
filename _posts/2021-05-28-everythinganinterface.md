---
layout: post
categories: [Philosophical, Design]
title: Interfaces On Everything
---

People in the .NET ecosystem are (or, at least, can be) overusing `interface`s and they shouldn't. That's the sentiment in [this Reddit post](https://www.reddit.com/r/dotnet/comments/n123a5/the_make_everything_an_interface_just_in_case/) which referenced [this tweet](https://twitter.com/JamesNK/status/1387592048254062592?s=19) which in turn referenced [this other Reddit post](https://www.reddit.com/r/dotnet/comments/mzxlrc/why_does_everything_have_to_use_interface/) on the same [/r/dotnet](https://www.reddit.com/r/dotnet/) subreddit as the first post. It's a bit of poetic irony, I think, that we're talking about excess indirection using excess indirection. 

The underlying concern here is that some people use `interface`s for *everything*, sometimes thinking that interfaces were somehow required to be "good design". This quote from James Newton-King on Twitter demonstrates some of this madness:

> The worst is when devs add an interface for every DTO so their `IUserRepository` returns `IUser`. You don't need to mock getters and setters!  

This string of pages has brought out some very interesting voices and some excellent points. I wanted to put out a bit of a longer piece about interfaces including when I use them and what I use them for. As always, your mileage my vary.

As a general reminder, the central question here is "why do people use interfaces on everything?" not "why do we use interfaces?". There are valid reasons to use interfaces and they can be an extremely powerful tool when used properly. It's the aggressive, *mindless overuse* of interfaces that is in question here. To narrow down on just one use-case, why would we use an interface on a DTO? I'll be assuming all the answers below are justifications for wild over-use, and not evidence that a person mis-read the question (I also won't be posting names here, because I don't want to shame a person who gave a good answer to the wrong question).

## Applications vs Libraries

First, I think it's highly important to differentiate a top-level application (a console program, a WinForms or XAML desktop app, a web service or web application, etc) from a reusable library. In an application setting, there are generally no downstream users who are interacting with the code directly. Instead they are typically interacting with the application through the user-interface or some other external surface. The application then must only be concerned with what it itself needs the code to do. You only need `interface`s in cases where there is actual required explicit polymorphism. You don't need to worry about what people *might want to do in the future*. 

Libraries, on the other hand, are intended to be consumed directly as code, and `interface`s are part of the surface with which a downstream user will interact. In the case of libraries it's often not possible to completely predict the use-cases of these downstream users, so a flexible surface should be provided to support many possible use-cases and composition of existing parts into new and novel forms. 

In an application I would almost never use `interface`s, and only when there is an explicit need for polymorphism to support the requirements. In a library, I would use an `interface` for many of the important objects with which the user is expected to interact.

## Testing and Mock Objects

> In order to use Moq for unit testing

Now I used to use mock objects for testing a lot, and I particularly liked the Moq library for that purpose. But at this point (spring 2021) I haven't used mock objects in unit tests *in years*. It's not that I haven't been writing tests. Far from it. I write as many tests today as I ever have. Over the years I've come to understand that mock objects as a solution for testing are just a *symptom of a bigger design problem*. That is, with better design there should not be a need for mock objects. When you see a mock object used, that's a sign that design somewhere can be improved. In other words, Mock Objects are a solution to a small problem, but a symptom of a big problem. Fixing the big problem almost always makes the small problem disappear.

Think of the example of an ecommerce website. We need to first check that we have sufficient inventory in the inventory system, then we need to process payment, and finally we need to notify the warehouse to prepare the shipment. We don't generally implement this by injecting our `IWarehouseNotifier` into our `IPaymentProcessor` and then inject that into our `IInventorySystem`. That kind of setup could be tested with mock objects, but in production when something goes wrong, you'd be scratching your head trying to figure out why an error with the warehouse has payment processing and inventory-checking method names in the stack trace! Instead of nesting these things together, we should be making them sequential and having some kind of `OrderService` or `OrderProcessor` call each system sequentially.

This sounds like we're just kicking the can down the road. After all we aren't using mock objects to test our `IPaymentProcessor` or our `IInventorySystem`, but now it looks like we're using mocks to test our `OrderProcessor`. What have we saved?

A principal that I try to follow here, and have mentioned on this blog previously, is this: **The cyclomatic complexity of a class should be inversely proportional to the number of dependencies of that class**. This fits nicely with another axiom I've seen before that **the cyclomatic complexity of the class should be proportional to the number of unit tests for that class**. If `OrderProcessor` has lots of dependencies, it should have extremely low cyclomatic complexity, because testing it is hard! The amount of code you write to create mocks for all dependencies makes your tests large and unwieldy. Remember that test code is still code, and all code has bugs. The more lines of code you have in your test, the more likely that the test itself will contain bugs! Then what piece of mind does the test buy for you? A class like this, low complexity and many dependencies, is best tested with *integration tests instead of unit tests*. This way you don't have to write huge amounts of setup code for your test, you can instead just use the existing bootstrappers and composite roots of your system to setup the dependencies that your system uses.  

`Moq` is a fine piece of software. It is well-considered and capably implemented for what it does. But, if you ever find yourself trying to use it in your own software you should instead go back and refactor the code so you no longer need it.

## Dependency Injection

> It allows you to properly use dependency injection

No, DI doesn't require interfaces, and you only really benefit from using them if you explicitly need polymorphism *right now* or if (in the case of libraries) there's a reasonable expectation that polymorphism may be used in the future outside of your control. If a particular abstraction has only a single concrete implementation and there's no expectation that a competing option will be developed and used with any frequency, you probably don't need an interface.

Every DI container or framework that I have ever used or heard of works just fine with concrete classes and doesn't require interfaces. Interfaces only give you the ability ignore which concrete implementation you're using if you have more than one (or potentially more than one). If you don't have more than one, you don't need an interface.

## Coupling

> To avoid tight coupling. Depend on contracts not implementations. Meaning you can change implementations without breaking your code.

Again, we're talking about putting interfaces on *everything*. In light of that, consider the case where we develop our own collection type as an implementation of a Max-Heap or a Priority Queue (which, by the way, I'm frequently annoyed that the .NET standard library doesn't provide one of these!). Is there any benefit to having an algorithm that explicitly depends on Priority Queue semantics, but decouples and uses `ICollection<T>` instead of just calling `new PriorityQueue<T>()`? Because if you have the abstraction there, some future developer who might not be as familiar with the algorithm might feel like there's flexibility to just use a `List<T>` instead of mapping to the specialized type, and then you get crappy performance or incorrect results or both! **Depend explicitly on the things that you need and which should not be changed**. An abstraction is a hint that something maybe can be changed, when you don't always want to give that hint. 

Further, there are times when coupling is ok. If you are in a bounded context, a self-contained submodule, or some other highly-cohesive nucleus of code, coupling together between classes which were designed together and are not intended to work apart from each other is fine. If I have a class which is private to a submodule or reasonably hidden from outside use, and which serves the specific needs of that submodule, I should be able to couple to it as closely as I want without anybody saying a damn thing about it. It's when you start interacting with code outside your submodule, or when governing interactions between submodules when you want more indirection.  In these cases `interface`s serve as a contract between modules, and allow the two sides of the interaction to evolve separately while still keeping the interaction itself stable. This is also a reason why your submodule should provide a *Facade* or some kind of *Service*, *Adaptor* or *Bridge* through which the rest of the application should interact with that submodule. Providing an agreed-upon surface for your submodule gives you freedom to design things as necessary behind that surface, without worrying about prying eyes or busy fingers going where they aren't supposed to go.

Related to that idea is this quote:

> I’ve never been able to swap an implementation without significant updates to the interface or call patterns anyway. Now I see stuff like `ISendGrid` instead of `IEmailService` and think that’s totally fine.

I know from experience that an email provider requires contracts and payment terms and all sorts of businessey stuff. You don't just switch email providers on a whim. So, on one hand it's unlikely that you'll need an abstraction here to give you the ability to change providers. On the other hand if calling code has to be aware that your email implementation is SendGrid, and has to take specific steps to support that implementation, then your abstraction is probably not well-designed. If you can't send an email from unrelated business code without that code having to be aware that the email is sent through SendGrid, you've done something wrong somewhere. A `SendgridEmailService` which implements `IEmailService` and has injected some code specific to SendGrid is fine. An `OrderProcessor` which doesn't has to inject `ISendGrid` and has to specifically interact with that object in a way that's specific to SendGrid and doesn't work the same with other email providers isn't fine. 

## Overriding Functionality

> But what if I want to swap out math? Of course I need to implement IAddNumbers…

This quote is a sarcastic one, of course. YAGNI, in my experience, is often over-used or abused to justify painful over-simplification. It's nice to have at least a general idea of the long-term goals in mind when you're writing software, even if those general ideas don't pan out exactly the way you imagined. Ignoring the future for the sake of radically over-simplifying today isn't smart development.

On the other hand, assuming that people might want to do a thing later, even if you can't possibly imagine a reason why, is foolhardy. A person may want to redefine the way addition works? Unless you either have an explicit use-case today, or have a reasonable assumption that it may be needed in the near future, don't create interfaces to support every little operation or idea in your code. Again, you have to be realistic and grounded in your prediction of what the future might hold. Be prepared for what reasonably might come your way, but don't waste effort preparing for all the most fantastic and ridiculous possibilities. Understanding where the line between the two is, comes from experience.

## So When To Use Interfaces?

Interfaces bring cost. There's a certain amount of cognitive load inherent in adding indirections. Worse still, vanilla Visual Studio 2019 doesn't include an easy way, that I'm aware of, to quickly find all implementations of a given interface in the current solution. Resharper does, but that's a costly add-on that many people don't have. Other extensions might do it, but I'm not aware of them. Even if this cognitive load is small, it's not zero and if you add many unnecessary interfaces you're going to come to regret it.

But you also don't want to use too few. Interfaces are an important tool and can enable very nice software design when used correctly. So what is correct? Here's a checklist I would propose using:

1. Do you now, or will you in the future with reasonable expectation, need polymorphism? Do you need to swap out implementations in some situations without calling code being aware of which implementation it is using?
2. Do you want to abstract away a particular decision, so that calling code isn't aware of the details? Hiding an `SendgridEmailService` behind an `IEmailService` abstraction may help you keep provider-specific details behind the abstraction and not let those details leak out into the rest of your system.
3. Are you thinking about a design that uses concrete inheritance? In those cases I would almost always use an interface instead, and delegate where code-sharing was necessary.

If these things sound like your use-case, by all means use an interface. Otherwise, it might be an unnecessary thing. **Complexity must be justified**, so you shouldn't just add another layer of indirection unless you specifically need it.

