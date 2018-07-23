---
layout: post
categories: [datalayer]
title: Problems with ORMs
---

This is a continuation of my previous post about [the data layer](2018-07-22_datalayer.md).

When I first started programming, I was completely enamored with ORMs. Specifically **Entity Framework**, which I found to be so simple to use and intuitive. This was back in the day when LINQ was still a new feature and LINQ-To-Entities was just blowing people's minds. EntityFramework was just obviously so much *better* than fiddling around with `SqlConnection` and `SqlCommand` and all that noise. You could replace strings of hand-written SQL ("blech!") with strongly-typed LINQ method chains, and everybody would just immediately see and appreciate the massive, immediate improvement. More readable and more maintainable! More resiliance to silly typos! A single model (especially after the Code First features landed) that could be used to generate both the data mappings *and* the DB schemas all in a single go! Who wouldn't want this?

Later we started to see things like WebApi demos, backed by EF, which showed how simple it was to setup a basic web application. So easy, in fact, that large swaths of the demo apps could be code generated in a matter of seconds! Just imagine how much work this technology could save us poor, overworked developers! Truly the ORM gods had smiled upon us.

The more I used ORMs in general and EntityFramework in particular, the more I started to notice some small cracks in the foundation, which later appeared as gaping chasms. The productivity benefits of Code First really weren't materializing because the people in charge of maintaining the databases, especially Production, didn't always have Visual Studio at their disposal and so weren't able to run migrations. Now developers are having to take time to generate change scripts from migrations and send them to the DBAs, which takes time and reveals some of the inherent messyness of the generated code approach. Then you start to get questions: Why are you setting the table up like this? Why don't you have an index on these columns? Why aren't you following our naming policies? The confused DBA, who happily writes SQL code all day long, doesn't understand why we, the developers, have abdicated this responsibility to an automated code generator which is getting things so obviously wrong.

(As an aside, I've found that most "naming policies" in databases are relics of a bygone era, before tooling and intellisense and arbitrary-length object names. So many of the standards which organizations follow no longer serve the intended purpose but you get locked in to them: We have dozens of databases which follow the old standard, and for no other reason than consistency we want the next database to just follow suit, etc)

And then one day you get an angry email from the DBA with a 10,000 line file of SQL attached asking "does anybody recognize this query? It takes several seconds to complete and is causing deadlocks or resource starvation, or other headaches!" No, nobody recognizes this huge pile of auto-generated gibberish, but after a day's investigation we've narrowed it down to an inconspicuous-looking line of LINQ code. Now the DBA, who has been biting her tongue for months about this "entity framework nonsense" goes to the CTO and demands all queries get moved to stored procs, which need to pass through a mandatory 9-step review process for every change. All those boosts to development speed you thought you had won with your ORM suddenly evaporate into thin air.

## The Problems

I don't mean to rag on EntityFramework, it's just the ORM that I have the most experience with. I started as a starry-eyed ORM zealot, slowly turned into something of an "expert" after years of seeing problems and learning how to fix them, and now have learned to dislike it completely. An ORM is just not the right solution to the problem, at least not for my needs.

The fundamental problem with ORMs is that there is an **impedance mismatch** between the objects of modern object-oriented software and the table/row structure of relational databases. Mapping an arbitary object hierarchy to a system of tables with foreign keys seems on the surface like a simple thing, but it never quite works right in practice. The ORM handles the mismatch by making a few assumptions about data, and hiding all the bits from the program and the database which don't fit neatly into those assumptions.

### The Problem of Identity

ORMs like to assume that there is a single row for every object, and a single object for every row. You can map a class to a table, and then specify a primary key property on the class which maps uniquely to the primary key column in the table.  This is a fine assumption for a simple data model and basic CRUD operations, but it falls apart for a lot of real-world scenarios.

Sometimes you want to materialize multiple different types of object from the same data, such as projections of the data, or only partially load  some columns and not others. Let's say I have a Product model and I would like to load out a subset of data such as the ID and Name only. EF and most other ORMs allow this kind of projection more or less. Now I want to take my `ProductIdAndNameOnly` object, make a change to the name and save that back to the DB. **Can't do it**. I need to load out the full product model from the DB first (which negates the savings from only loading out the projection in the first place), make the changes, and then save the whole object.

"But wait!" I can hear you saying, "You can just instantiate a new Product object without loading it first, attach it to the DB context, update your Name field, and make a few metadata calls to let the ORM know that this object is an update and only some fields are updated, and then commit the changes. It's easy!"

Easier than writing a simple statement to `UPDATE Name WHERE Id = @id`? But ORMs don't let you just do that, because of cached object references. Any modification you make outside the ORM might not update the object cache and the next "load" may leave the application in an incorrect state.

EF will also run you into lots of "cannot insert duplicate primary key" errors when you try to save an object graph which, unbeknownst to you, contains copies of entitites which look right but aren't accounted for in EF's internal bookkeeping. EF thinks it's supposed to be an `INSERT` because it hasn't seen that object reference before, even though it has an ID of an existing row. This issue severely complicates operations such as caching of data which is frequently loaded and infrequently updated.

### The Problem of Symmetry

CQRS is a thing, and even if it wasn't there are still plenty of times when what we need to load out of the database isn't the same as what we need to insert into it. We do need mappings and projections. We do need groupings. Databases offer things like `GROUP BY` and window functions which ORMs don't easily expose (`GROUP BY` clauses in SQL code are very simple compared to the `.GroupBy()` LINQ method which is a readability headache on it's best day). There is also the `.Aggregate()` method, but you have to keep track of which kinds of aggregations are even supported by the ORM and which ones will cause a runtime error because the expression parser can't handle them.

The problem of symmetry is that the kind of model you want to work with in your application isn't the kind of model you want to work with in the database. This is why your system probably contains a separate Data Model from your Domain Model, and why you have a fancy Data Mapper between them. The data you want to insert isn't always the same as the kind of data you want to query. For an ORM like EntityFramework, these kinds of differences are anathama to normal operation.

I have a Product. Product has normal fields like Name and SKU, and also has a list of `ProductAttributes` which are name-value pairs of configuration options and other metadata. We have an attribute `{ color: red }` and `{ size: x-large }`, but then something like `{ countryOfOrigin: USA }`. It's easy enough to load out the product model using LINQ:

```csharp
var product = context.Products
    .Include(p => p.ProductAttributes)
    .ToList();
```

A requirement comes in saying that we want to query only products which are Made in America for a promotion. If a ProductAttribute has a string name and a string value, this query is easy enough:

```csharp
var product = context.Products
    .Include(p => p.ProductAttributes)
    .Where(p => p.ProductAttributes
        .Any(pa => pa.Name == "countryOfOrigin"
            && pa.Value == "USA"))
    .ToList();
```

In this model, however, you can't strongly-type your attributes and you can't leverage your compiler or serializers to do validation of types for you. What we would really like to have in our OO code is something like this:

```csharp
public enum CountryType {
    USA,
    Canada,
    ...
}

public class CountryOfOriginAttribute : ProductAttribute
{
    public CountryType Country { get; set; }
}
```

With a strongly typed object like this, you'd never be able to set a nonsensical attribute value such as `{ countryOfOrigin: red }`, the language simply won't allow it. But if we did this, how do we map our query to EF?

```csharp
var product = context.Products
    .Include(p => p.ProductAttributes)
    .Where(p => p.ProductAttributes
        .OfType<CountryOfOriginAttribute>()
        .Any(pa => pa.Value == CountryType.USA))
    .ToList();
``````

This and more complicated queries like this, don't work for me, and this is simple compared to some of the queries you run into with real-world data. What is your `CountryOfOriginAttribute` has child objects that other attributes don't have, such as details related to tariff and duties and port-of-entry? How do you `.Include()` that in your query? You end up have to break out multiple queries, but EF can't batch them into a single DB connection, so you're hitting the DB twice and paying a penalty for it.

The "standard" design is to have a Domain Model and a Data Model separated by a Data Mapper, and that mapper is going to get more and more complicated over time as the two models start diverging. Plus the code the load out your Data Model from the DB is going to get more and more complicated; a huge tangle of `.Include()` and `.Where()` and multiple queries that you have to join together in the application.

(As an aside, WebApi in VisualStudio still gives you the option to auto-generate controllers "backed by Entity Framework" where a single model is used not only for the data model and the domain model, but also as the web request DTO. The severe violation of the Single Responsibility not withstanding, you now have a model class serving two different masters: Consumers of your API contract need to be aware of the structural details of your database, and you can't change either your API contract or your database structure without perfect agreement from both sides! I can't think of a worse nightmare.)

### The Problem of Obscurity

Obscurity is not a design goal of ORMs, but it is often envitable. Auto-generated SQL code, even from sophisticated modern code generators, can still become a huge mess. The EF team has made a design decision that queries should be converted into a single result set which can be handled with a forward-only cursor. This way you can consume the result `IEnumerable` in a streaming fashion without necessarily needing to `.ToList()` the results first and eating up a huge chunk of memory. This is definitely a benefit in some cases, but I wish we had the ability to turn that off and say "the results of this query won't be streamed" so that EF could spin up a more sane code generator. But then EF would have to provide two code generators, which seems like extra work for them because they already have one code generator which test well-tested to work correctly.

Not only are auto-generated queries typically unreadable, but they are also un-tuneable. You can't optimize a query if you can't edit it. You can try to rearrange the LINQ statements in hopes that it produces a better query, but you can't hand-tune the SQL query yourself. When you run into a performance problem with a query, your only options tend to be "just live with it" or "get rid of EntityFramework". The first amortizes headache over time, the second creates a huge headache right now.

### The Problem of Application Primacy

The ORM doesn't do work in the DB. It doesn't do in-place bulk updates, or in-place bulk deletes except maybe in a few special circumstances. To make a change in the database the ORM wants you to query the objects you want to modify, load them whole-hog into the application, make your changes in memory, and then commit all your changes back to the database. This is hugely inefficient for a whole set of common problems. Being able to delegate some bulk processing work to the database is a huge benefit of database technology and not leveraging that in your application because your ORM decided not to expose that functionality is a huge lost opportunity.

(And don't even get me started on `OptimisticConcurrencyException` which is both very difficult to reproduce in local testing and doesn't contain enough useful information to track it down when it does appear in your log files. If I wanted the ORM to not update the data I would have called `.SaveChangedAndSometimesRandomlyThrowExceptions()` but I didn't call that, I called `.SaveChanges()` instead. Fuck `OptimisticConcurrencyException` in it's stupid fucking face.)

## The Power of SQL

Before I go any further, I want to say that I'm not a particular fan of the SQL language or of relational databases in general. Too often I see people trying (and often failing) to do complex work in an SQL script which would be trivially easy in a proper programming language, or trying to jam data which isn't inherently mappable to tables/rows into SQL databases. I also feel that, for most applications, a document store of some sort would be a significant improvement over a relational database for a whole host of reasons.

That said, once you've committed to using an SQL-based relational database in your application it is completely foolish not to leverage the power and benefits of that platform. Using a sub-optimal storage engine and then refusing yourself the benefits that engine provides is stupidiy. SQL, at least in it's particular domain, is a very powerful and very expressive language.

If you want to get down and dirty with the raw SQL you typically have two options: Micro-ORMs and the `System.Data.SqlClient` namespace itself.

### Micro-ORMs

There are a few micro-ORMs out there, Dapper is one that is mentioned frequently, which avoid the complexity and much of the problems associated with EntityFramework and NHibernate. While these are superior for many use cases, I still find myself frustrated by some of the assumptions made and the limited abilities exposed. For instance, not having enough control over how result sets are mapped back to result objects, or not having control over query caching.

### `System.Data.SqlClient`

While I'm talking about alternatives, the raw power of the `System.Data.SqlClient` namespace is hard to argue with. Unlike the Micro-ORMs you do get full control over everything, and unlike regular ORMs there are no assumptions made which get in your way. The problem with the tools in this namespace is lack of usability. Everything you try to do with `SqlConnection` and `SqlCommand` and their friends is a total slog: `using` blocks inside `using` blocks and loops within loops. You have to do all sorts of manual error-checking because the exceptions these classes throw are completely worthless and devoid of helpful information. Even an operation as simple as "read the value from a column by column name" is much harder than it needs to be and prone to significant errors. Why they have a separate `DbNull` instead of just casting to `null` or even `default(T)` in some cases is a constant source of annoyance.

## Up Next

I'll continue this discussion in the next post.