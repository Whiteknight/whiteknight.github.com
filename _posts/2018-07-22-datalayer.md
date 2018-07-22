---
layout: post
categories: [datalayer]
title: The Data Layer
---

The **data layer** is such a complicated beast, even though it serves a relatively simple purpose: provide persistant storage the application data. There are a number of different patterns that can be leveraged, in theory, giving the developer great flexibility. On the other hand there are a few common frameworks (ORMs and Micro-ORMs, etc) which provide some of these patterns but also impose steep limitations. The trade-offs you have to make just to get started with persistant storage can have severe long-term ramifications on the design of your entire application.

Your application might not be large or sophisticated enough to warrant having a separate "layer" for data concerns. You save a little bit of organizational effort just keeping everything together, but you run into serious risks if your application does end up growing to a significant size and you lose the opportunity to practice some important concepts in application architecture. I would always try to do at least some basic separation and abstractoin with my data layer, unless I was in some kind of extreme circumstance (and I have a hard time even imagining what that circumstance might be).

## How To Organize It

I tend to like to keep my data concerns separated from my core domain code. For small little projects this can mean a `Data` folder and a `Data` namespace. For larger projects I like to create a separate `Data` assembly to hold all necessary code and dependencies. This allows me to *isolate the dependency* on any ORM or other data access libraries, and keep all of the complexity of the data layer separated from the rest of the system.

Depending on the number of data sources, I may even have more than one `Data` assembly. Maybe `Data.Sql` and `Data.Elastic` assemblies, each focusing on a different storage engine. If you are caching data results, in Memcache or Redis for example, you could build that logic into your existing data assembly or you could create a separate `Data.Memcached` assembly and compose your data pipeline up top in your **composition root**. If it's a small thing and doesn't have a lot of complexity, just build it all together.

Also, I would separate databases which my application "owns" from databases which have other owners but I need to access directly. (Direct access to a database which your application doesn't own isn't a good idea, but some organizations aren't mature and sophisticated enough to wrap every database in an application, or expose all the endpoints you need when it is wrapped, but that's a different topic for a different day). So I could end up having assemblies for `Data.MyDb.Sql` versus `Data.TheirDb.Sql`. That way when one of my dependencies makes a breaking change, my team knows exactly where to go to find and fix all the references.

## Interacting With Core Logic

Your core business logic can consume **separated interfaces** for the data layer, be they in a form of a data-layer "service" or some kind of data "gateway". Going through an abstraction like this forces you to clearly define what operations your data layer allows and give those operations descriptive names. It also gives you an opportunity to think (eventually) about optimization: data operations which are frequently performed together can probably be combined into a single batch operation.

The Data Layer should provide operations which work on Domain Objects. Your core business logic (domain objects, services, etc) should be able to pass domain objects around and not worry about mapping to specialized formats. Mapping to and from the business domain and the persistance domain in the responsibility of the data layer.

## How It Works

At the highest level, the data layer needs to do a few things with every request:

1. Map the request from the external (business) domain to the data domain
1. Perform the data operation
1. Map the results from the data domain back to the external (business) domain.

With a document data store, there might be no mapping or very little mapping. With an SQL database your data model typically needs to conform pretty closely with the structure of your database and mapping between that and your domain model might end up being significant.

Consider the case of LINQ-To-Entities with EntityFramework. Mapping to and from models isn't difficult, but mapping `Expression<Func<MyDomainModel, bool>>` in a `IQueryable<T>.Where()` query to `Expression<Func<MyDataModel, bool>>` might be nigh-impossible. When you use an ORM like EntityFramework you're either forced to abandon the Domain Model and do everything directly with the Data Model or you're forced into a rigid abstraction where your data layer doesn't expose fancy `IQueryable<T>` methods above a certain point in your stack and every single separate query, minus some property-based parameterization, requires a named method. I'll talk more about ORMs in general and EF in particular in my next post.

## Up Next

I've been working on a new library for doing some data-layer stuff. I didn't want to just post about that out of the blue and not explain why I've been heading in that particular direction. I started writing up some of my complaints with the existing state of things and some of the explanations for my ideas, and I figured it was getting longer than a single blog post. Now I'm stuck with a whole series of blog posts. Next time I'll talk a bit about ORMs, and after that I'll talk about some of the patterns and concepts in data layer design and why I like or don't like them.