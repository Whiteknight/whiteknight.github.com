---
layout: post
categories: [Design, datalayer]
title: On DSL Query Languages
---

I saw [this blog post about DSL Query Languages](https://erikbern.com/2018/08/30/i-dont-want-to-learn-your-garbage-query-language.html), and the author makes some pretty good points.

## ORMs

One point that I particularly like because it reenforces my thesis about [CastIron](/2018/07/25/castiron.html), is this bit:

> Take ORMs. Their alleged benefit is they cut down development time. But instead of writing SQL which everyone knows, I know how to scroll back and forth in some ORM documentation to figure out how to write my queries. On top of that, I have to spend time debugging why the ORM translated my query into some monstrosity that joins 17 tables using a full table scan.

My thoughts exactly. Later down he says:

> Let’s dispel with the myth that ORMs make code cleaner. Join the embedded-SQL movement and discover a much more readable, much more straightforward way to query databases.

I didn't even realize there was a movement. Sign me up. I'll bring drinks and cookies.

## DSL Query Languages

The author says that instead of creating a new single-purpose query language for every new storage technology, they should just use SQL. He says of SQL:

> It’s a language everyone understands, it’s been around since the seventies, and it’s reasonably standardized. It’s easy to read, and can be used by anyone, from business people to engineers.

I agree with this for the most part. It's very common that I wish MongoDB or InfluxDB or other data stores had an SQL or SQL-like query language that we could just use without having to learn some new custom query language. There are a few small caveats, however, which stop my support from reaching a full 100%:

1. Please keep "business people" out of the database. It's hard enough to get databases to perform well without naive queries locking the whole thing down.
1. SQL has become a relatively large language, so we're probably talking about just the Data Manipulation Language (DML) subset. Even with just this language subset, there are a number of clauses and features most non-RDBMS databases won't utilize, so it's probably a pipe-dream to expect we'll be able to take the same exact SQL statements between different storage engines. Even among SQL-based RDBMS engines like Microsoft SQL Server, Oracle, PostgreSQL, MySQL and SQLite, differences in dialect can be significant and can limit code portability.
1. SQL was designed specifically for RDBMS engines and evolved together with them. It really was never designed to cover the non-RDBMS features or data storage ideas of other systems. It may have plenty of uses other than this, but it's not necessarily of universal application (though I do agree that it is very general and would be suitable for many applications where it is not currently used)
1. For a system which has enough unique ideas and limitations, such as InfluxDB or ElasticSearch, a custom query DSL might do better to expose the particular abilities of the language without having to leave common clauses for the existing SQL language undefined.
1. Much of the "standard library" of functions and stored procs that ship with various DB engines wouldn't work or have any meaning in a different storage engine. Hell, `NEWID()` doesn't even work in SQLite, so portability of less-used functions to even less-similar storage engines is a dubious proposition.

A document store like MongoDB could probably be served by SQL or a nearly-SQL dialect, though things like column names in the SELECT list don't really work because what we want in most accesses is just to return the entire document as a single unit. In Mongo it might be much more efficient and idiomatic to select the entire document where in SQL Server it's typically better in both accounts to just select the specific columns required. Using a single query language to represent both of these two different use-cases might cause a bit of a headache.

Most ElasticSearch queries want to return not just a page of results, but also aggregated statistics of the entire result set (or a related result set) to facilitate drill-down.

InfluxDB maintains a strict separation between the "normal" data columns and the "tag" columns and allows only certain operations on each. It also allows, but is generally not optimized for, `DELETE` and `UPDATE` operations on existing records.

I bring up these few examples just to show that, while SQL is pretty general, it is probably insufficent for all cases. Even optimistically the best we would end up with is a close dialect of SQL which might be enough to decrease the mental load of the development team. It's like saying that C# is a standard language, but there are some differences, often significant, between programming an IIS website and a console application.

## Agreement

Yes, I do wish that when developing a new storage system (and even some webservices) that developers looked at SQL for inspration first, and only moved away from that slowly and with great care. Even if they ended up with an SQL "flavor" that was as close to the original as possible, it would still be a good thing.
