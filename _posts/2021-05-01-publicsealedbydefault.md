---
layout: post
categories: [Philosophical]
title: By Default
---

Mark Seemann has been writing a very [interesting series of posts about code](https://blog.ploeh.dk/2021/02/22/pendulum-swings/), especially about default class modifiers. Several of these posts in particular have caught my attention, some because I disagree with them and others because I whole-heartedly agree.

## Public By Default

Mark argues that new classes should be marked [internal by default, public by choice](https://blog.ploeh.dk/2021/03/01/pendulum-swing-internal-by-default/). This is the one I disagree with, though not for the reason he might think. Mark says some of these things about why people like to mark things `public` by default:

> What little I knew about encapsulation led me to believe that the more internal my code was, the better encapsulation it had. 
> When I discovered test-driven development (TDD) (circa 2003) all my classes became public. They had to.
> Yes, it's technically possible to test internal classes in .NET, but I don't believe that you should.
> You're testing something you care about.
> I decided, however, to keep this API `internal`... the behavior is covered by integration tests. I feel free to refactor.

I'm cherry-picking quotes here, the original post is full of lots of information so I suggest you go over there and read it in it's entirety if you haven't already.

People think that encapsulation and information-hiding require classes to be `internal` by default, to not show your cards to other players at the table. But this is put at odds with the drive to unit-test everything, and tests really require code to be `public`. (And I agree whole-heartedly with that quote in the middle, if you're specifically limiting a class to be `internal`, and then immediately disregarding that access modifier in your own tests, that's a major red flag. People who need to use your code downstream are going to use your test suite as an example of *how to use your code*, and if you're showing them that you expect encapsulation to be broken, upi are inviting in your own misfortune).

It's not that you unit test things you *care about*, but instead you test to create a contract with your users: "This is how the code is expected to be used. Here are the guarantees I'm making that the code works as advertised. This is how I know that the code is bug-free and suitable for public release. This is how I know things won't break in the future."

If you look at [StoneFruit](https://github.com/Whiteknight/StoneFruit) for example, most of the tests in the test suite are end-to-end integration tests because that's how users are expected to interact with the code. Instead of marking some classes `internal` to hide information, I do it by namespace. The [root namespace](https://github.com/Whiteknight/StoneFruit/tree/master/Src/StoneFruit) has only a small handful of classes and a large number of abstractions. If you want to get into more advanced use-cases, you can start `using` more and more deeply nested namespaces. You choose your own level of involvement, and the deeper you choose to go, the more power but also the more responsibility you will have.

The last line of quote above shows why Mark believes in `internal` by default: It gives freedom to refactor and encourages more integration testing than unit testing. I agree, these are positive outcomes from this approach.

Despite that, the reason I believe in using `public` by default actually has nothing to do with testing (although, ease of unit-testing is a nice benefit, especially for integration scenarios where there are so many configuration options that testing all combinations would be improbable). Instead, it's the combination of two important principles:

1. All abstractions leak
2. Prefer composition over inheritance

When I'm writing a library, I try to provide abstractions and facades to provide an easy on-ramp for most users. The common cases should be easy, the beginner use-cases should be dirt simple. However I also like to provide all the individual pieces so power-users can compose their own solutions from the same building-blocks that I used. You cannot reuse old code to compose new solutions if you cannot access the old code. I have been bitten so many times by libraries whose authors don't understand this principle, and I've had to go to extraordinary lengths to duplicate functionality which already existed but was hidden behind a thoughtless access modifier. So many times people mark something `internal` or `private` by force of habit, when that piece would be perfectly usable by downstream users. Now when you want to access those parts, you have to open a ticket and make an argument for it, when it should have been available from the start! If you want users to be able to compose their own solutions, you must provide them access to the building blocks necessary to compose them.

I make all classes `public` by default and only mark something `internal` when it has no reasonable use in downstream code and when the mere existence of the class or method will cause confusion, invite misunderstanding, and create unnecessary headache. 

I guess my approach to encapsulation could be called something like "best effort encapsulation", or "need to know basis". If you don't need to know the implementation details, the system shouldn't force you to care about them. In the rare (but inevitable) case where somebody *must care about the internal details* they should be accessible in some fashion.

## Sealed by Default

Mark also says classes should be [sealed by default](https://blog.ploeh.dk/2021/03/08/pendulum-swing-sealed-by-default/). My discussion of this is going to be much shorter: **I agree with it in it's entirety**. Classes should be `sealed` by default, and I lament the fact that the default class template in Visual Studio doesn't add this modifier (or provide an easy way to amend the template to include it). I did modify my own template, but that's a huge pain in the butt to find the correct file and make the necessary modifications to it. Visual Studio absolutely should include an editor for templates, and should save modifications to your MSDN account so those changes follow you from computer to computer. 

Inheritance *is* evil. I believe you should prefer composition over inheritance in almost all cases. Make sure that your composable building blocks are properly `public` so people can make use of them. Sealed by default is what it should be, and only when you explicitly say so should a class be able to be inherited.

## Pure by Default

The third point he makes, and the last one I saw before this post was published, is about making methods [pure by default](https://blog.ploeh.dk/2021/03/15/pendulum-swing-pure-by-default/). That is, the code should have no side-effects and it's output should be entirely determined by the input.

Pure code is great. It's easy to test. It's easy to compose. Remember when I said above that you should prefer composition over inheritance? You can that by trying to make as many of your methods and classes pure as possible. You want to try to separate out the impure code from the pure code. Mark recommends [a sandwich architecture for this](https://blog.ploeh.dk/2020/03/02/impureim-sandwich/): Use impure code to load data and user input, pass the data to pure code for manipulation, and then pass the results to impure code to output it and cause side-effects. This is exactly how I like to organize my Application Service Layer, as an orchestration of separate pure and impure code. Everything below the application service layer should be strictly pure or strictly impure if possible (with much more code falling in the "strictly-pure" pile than not).

## History Lesson

I've covered some of these topics in prior posts. Notably I've talked a bit about [Access Modifiers and Inheritance](http://whiteknight.github.io/2019/10/05/csharpmodifiers.html). If I could wave a magic wand and change C# syntax in any way I chose, I would make these changes:

1. Classes must have an explicit access modifier, no more `class Foo { ... }` which implies `internal`. You would have to either explicitly specify `public` or `internal`
2. I'd allow `private` modifiers on classes, which would only make them accessible to other classes in the same namespace. This would help a lot with creating namespace-based submodules with proper encapsulation
3. I'd make all classes sealed by default, and require some kind of new keyword such as `unsealed` to mark a class as being inheritable. (And then I would probably make use of `unsealed` generate a compiler warning).

Backwards compatibility being what it is, these kinds of changes will probably never happen in C#, even though I think they would be an improvement.

## Pendulum Swings

I've remained a little bit more consistent than some people about this over time. Maybe I'm right about it all, maybe I'm just a stubborn mess. Smarter minds than I will eventually decide how software should be written, and a lot of my writings on this blog will turn out to be total crap. But, until that point, I'm sticking to my guns. Most classes should be `public sealed` by default, concrete inheritance should be avoided, and effort should be taken to separate pure and impure code (and the former should significantly outweigh the later, where possible). The best software I've seen and the best that I've written follows these rules, and the worst software I have ever seen breaks them all. Maybe this is a coincidence. Maybe I haven't seen enough software and there are too few data points to draw a real conclusion. Maybe. But until I see otherwise this is what I'm going with.