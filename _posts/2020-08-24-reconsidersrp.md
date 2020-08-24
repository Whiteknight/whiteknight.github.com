---
layout: post
categories: [Design, Theory]
title: Reconsidering the Single Responsibility Principle
---

There's a problem with the **Single Responsibility Principle** (SRP): *There's two of them*, and people (myself included) confuse the two all the time. The first is the original version formulated by Robert C. Martin ("Uncle Bob"), which is all about change and people. Most developers in my experience aren't familiar with this version, don't understand it or don't know how to apply it. The second is the more colloquial understanding which, while not supported by any authoritative source, seems to be well-known by and leads to some very good design results when it is applied.

I frequently find myself thinking and even talking about SRP in terms of what I wish it was, not what it actually is. I wish it was a principle that provided clear **guidance**. How do we draw boundaries in a piece of software in a way that improves the understandability, maintainability and flexibility of it? I believe this is what the  idea of "singleness" at the heart of SRP could provide, if we could just find a better way to formulate it.

## Original Formulation

Uncle Bob first formulated SRP in the 1990s, though the first I saw it was in his book [Agile Software Development](https://www.amazon.com/Software-Development-Principles-Patterns-Practices/dp/0135974445/) which was published in 2003 (He has several books, all of which are good reads for most programmers and I highly recommend them if you're looking to improve your game). He has also written [about it on his blog](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html). His most well-known published formulation of this principle is:

> A class should have only one reason to change.

He then says a few other things that are interesting:

> There is a corollary here. An axis of change is an axis of change only if the changes actually occur. It is not wise to apply the SRP, or any other principle for that matter, if there is no symptom.

> Conjoining responsibilities is something that we do naturally. Finding and separating those responsibilities from one another is much of what software design is really about.

So right off the bat I see a problem here: Uncle Bob doesn't want us to apply SRP to code which isn't changing, but he also acknowledges that it's a goal of software design to separate responsibilities. Are we only expected to separate responsibilities in code which is in frequent flux? Is code which is being written but has no obvious impetous to change in the future exempt from good design?

In his chapter on SRP from *Agile Software Development* he also talks about reasons why we might want a single class to contain two or more responsibilities, despite the name of the principle being "Single Responsibility" and his mention above that the goal of design is to separate responsibilities. His example is a `Modem` class which may need to implement both connection management (`dial()`, `hangup()`) and data flow (`send()`, `recv()`).

In his blog, Martin does some more explanation:

> The Single Responsibility Principle (SRP) states that each software module should have one and only one reason to change....However it begs the question: What defines a reason to change?

> ...These questions can be answered by pointing out the coupling between the term “reason to change” and “responsibility”.

> ...a better question is: who is the program responsible to? Better yet: who must the design of the program respond to?

> This principle is about people.

By talking about the principle this way, it starts to sound more like a prescriptive version of [Conway's Law](https://en.wikipedia.org/wiki/Conway%27s_law), choosing to actively design the system to mirror the design of the team of stakeholders, instead of allowing the system to approach that design over time. This isn't a bad idea *per se*, but it wouldn't be considered a new principle so much as an extension of the existing law.

In his blog Uncle Bob lists a few of his inspirations for creating SRP. He lists Djikstra's work on *Separation of Concerns*, and he mentions work done about *cohesion* and *coupling*. He also quotes David L. Parnas:

> We propose instead that one begins with a list of difficult design decisions or design decisions which are likely to change. Each module is then designed to hide such a decision from the others.

This, of course, is *encapsulation*. Taking these inspirations together you might be forgiven for thinking that SRP was about **separating concerns to improve cohesion, reduce coupling, and hiding information through encapsulation**, which actually seems like a good principle by itself. This is why it's so confusing when Uncle Bob talks about SRP being about "change" and "people". (Parnas' quote only talks specifically about encapsulating "difficult" decisions or "decisions which are likely to change", but I'm going to extend my meaning of "encapsulation" in discussion below as information-hiding for all types of decisions, including the simple or static ones).

## Colloquially

Now that we've looked more closely at the original formulation, I'd like to list off a few of the colloquial versions of SRP that I've seen (online, at work and in job interviews). Here is a list of things I've heard and a few of the first explanations I find when searching Reddit:

* A class should only do one thing
* A class should do one thing and do it right
* [Only do the right thing](https://www.reddit.com/r/softwaredevelopment/comments/iehjwe/solid_principles_in_5_words_or_less/)
* [an object should only have a single conceptual role](https://www.reddit.com/r/softwaredevelopment/comments/iehjwe/solid_principles_in_5_words_or_less/g2gi8jo?utm_source=share&utm_medium=web2x&context=3)
* [Ensure your class or object only does one thing at a time](https://brabble.fm/podcast/the-rabbit-hole-an-inside-look-into-software-development/162-solid-single-responsibility-principle)
* [A class should have a single responsibility and this responsibility should be entirely encapsulated by the class.](https://blog.ndepend.com/solid-design-the-single-responsibility-principle-srp/)

There really does seem to be a concept of "singleness" expressed here, although a single one of what, exactly, is not clear in any of these nor consistent between them. Notice that none of these formulations mention, express or implied, either *change* or *people*. I would like to reiterate that all of these, despite not being the same principle as what Uncle Bob first formulated (though, there is some clear overlap), have value in helping guide a developer to producing better code. People repeat them because they work.

## Vagueness

The one complaint I hear most about SRP, in either of it's versions, is that it is too *vague*. It's not clear what, exactly, comprises a "responsibility" or a "reason to change". It doesn't help that Uncle Bob is quick to point out an example class which he believes should have two responsibilities (connection management and data transfer in a single Modem class). By being so vague, and being open to so much interpretation, it's hard to get consistent results. In fact the opposite is quite easy: By interpreting the words favorably, you can almost justify any design you want. While I do firmly believe in the benefits of designing a unit of code with singleness of purpose, I will definitely admit that the vagueness of existing formulations is a serious issue.

## Units of Code

I've used the term "unit of code" a few times so far and will be using it more. I'm using the term to describe any self-contained piece of code around which a clear boundary can be drawn to separate it from the rest of the system. For example, the code inside a method is clearly separated from the rest of the system by the boundaries of the method itself. Code inside a particular class is likewise clearly separate from code outside that class. And you may have classes within a module/subsystem/bounded-context being clearly separate from other classes in other groups. At the very top of the hierarchy, there are modules grouped into programs, services and libraries, and these things grouped into even larger structures as well.

At each higher level of organization, abstraction should go up, details should go down, and the effort to make change increases. Each one of these should represent an appropriate singleness. At the lowest level it may occasionally be useful to talk about a single line of code as a self-contained unit (separated by newlines and, depdending on language, semicolons or parenthesis). Principles like SRP being mostly used to improve understanding and maintainability, and individual lines already being already maximally understandable and modifiable, it is not usually necessary to go down to this level. I'm only mentioning it for completeness.

Large units are composed of smaller units, which I will call "children". Lines are children of the method, methods children of the class, etc. Changes to the design of a single unit may require changes to all the children of that unit. For my purposes here, making a change to a unit which necessitates changes to child units, will be considered a single change.

## Design Decisions

I use the phrase "design decisions" a lot. There are many different types of decisions which all fall under the heading of "design". For example, there are the decisions related to the business requirements as given by the business stakeholders. There are also decisions about structure and flow which may come from an architect or other technical leadership. Then there are decisions about implementation which typically come from the developers themselves. Each of these should be considered separate, even when one decision influences another. For example, a certain business decision which asks for data to be sorted and quickly accessible may imply the implementation decision to use a binary tree. Despite the two decisions being closely related, it's still probably a good idea to separate out the implementation of the binary tree (low-level of abstraction, highly reusable) from the business logic which makes use of that tree (higher level of abstraction, not reusable).

It's also worth mentioning that as requirements change over time, the design decisions made must be re-evaluated. A new decision which modifies or extends an older decision superceeds that older decision. For example, if we decide to store a database connection string in a configuration file, but then later decide to store the credentials in a separate secure location and compose the connection string at runtime, we have a new set of decisions replacing the single older decision.

We can identify our decisions through an iterative process, starting with what we know we have (or what we know we need to have) and repeatedly asking "how are we going to do this?"

We start with a connection string accessor abstraction, charged with retrieving a connection string.
* How does the accessor get the connection string? It assembles secret and non-secret parts using a template engine
* Which template engine? Liquid
* How do we retrieve the secret credentials? From the secret vault offering of our PaaS provider
* How do we retrieve the non-secret connection string information? From the configuration file.

A design decision, in short, is the answer to a question such as "how...?" or "which...?". Other questions such as "where...?" or "when...?" may be a design decision or maybe a configuration parameter, depending on implementation.

## Other Related Ideas

The principle of **Don't Repeat Yourself** (DRY) seems like it fits very neatly into this discussion. If there is a one-to-one relationship between design ideas and units of code, there would be no repetition. But I would have to take one caveat into consideration: Two identical pieces of code which represent different ideas, serve different masters, and may change for different reasons, are not duplicated. That is, two ideas may have forms which are coincidentally and temporarily identical, but they are not the same idea and the code should not be considered a repetition. It isn't repeated knowledge, it is only similar form.

**Conway's Law**, as I've already mentioned, is salient. As in my mention of DRY above, the design decisions which we need to encapsulate away must incorporate information about the forces which may cause those decisions to change. Taking this information to account, over time, will lead the design of your software to mirror the structure of your team.

There's also the idea of **Composability** which is implied by things like the **Open/Close Principle** (It would be preferred to create new solutions by composing and configuring existing units than to modify existing units) and the maxim to "prefer delegation over inheritance" (despite that preference not having a fancy name). A single unit which is small and focused on a single purpose can be composed together easily. A unit which incorporates many ideas and responsibilities may not be easy to compose because a solution will be forced to accomodate ideas which are not germaine to the solution.

## Basic Statement of Principle

Let me list out a first draft of a new formulation of this idea of "singleness", which I think is an improvement over both the other formulations I've spoken about because it uses more precise language.

> A single unit of code should correspond uniquely to, and encapsulate, a single design decision

**Lemma 1** (*Completeness of Decisions*): A "design decision" includes not only the result of that decision, but must also account for the level of abstraction at which the decision is made and the forces which shaped that decision and may cause it to change in the future. Multiple design decisions at different levels of abstraction or which are shaped by different forces cannot be considered as a single decision.

**Lemma 2** (*Single Point of Change*): Should a design decision change, there should be a single unit of code in your solution in which that change should be made (including children at a lower level of abstraction).

**Corollary 1**: A unit of code which does not represent any particular design decision can be removed or merged into other units.

**Corollary 2**: A unit of code which represents multiple separate design decisions should be considered for separation into multiple units.

**Corollary 3**: Multiple units of code which have high cohesion and represent parts of the same design decision should be considered for merging into a single unit.

Notice there are a few important changes here from other formulations of SRP: The one-to-one uniqueness relationship between the unit and the decision is not often expressed and has, I believe, real benefits. The linking of the unit of code to a design decision in particular (instead of a "responsibility", "purpose" or "reason to change"), and being more precise in my definition of "design decision", I think the principle becomes more concrete and applicable. I recognize there is still some vagueness here, but I firmly believe this is a step in a better direction.

## Summary

So that's where I'm leaving this topic for now. If this thing needs a name I might suggest something like "Single Decision Principle", which I think is a pretty accurate simplification of the wording of the principle itself. Otherwise, we could just cram this into the large soup of interpretations for the existing SRP and call it a day.