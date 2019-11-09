---
layout: post
categories: [projects]
title: SqlParser
---

I have been doing some work on a few side projects recently, I'll be sharing some of what I've been doing in the next few posts. First up is a parser library for (a subset of) the SQL language. I started this project for two reasons:

1. I wanted to practice some recursive descent, which I haven't used in years and
1. I wanted to better learn the SQL language.

This is a toy project right now and I don't want to act like I have serious long-term plans, but these kinds of things do have a way of spiralling out of control. You add one feature for fun, then another, then another, and suddenly you have something usable that you want to update to Nuget. You know how things are.

It started as a branch in the [CastIron repo](https://github.com/Whiteknight/CastIron) but I have since decided to move [to it's own repository](https://github.com/Whiteknight/SqlParser) so I wouldn't polute my "production-ready" code with something of much lower quality.

## SQL Dialects

It's worth mentioning up front, for anybody who isn't aware, that SQL has a number of dialects. The SQL standard itself gets pretty fuzzy around the edges, and a lot of details are left up to the particular implementation. In addition, almost all implementations add a number of "language extensions" which are features that they support but the SQL standard itself doesn't specify. Often times, the same exact extension feature will be added by different vendors in different ways.

The SQL Parser that I've been working on is currently aimed at T-SQL from Microsoft's SQL Server. It's the one I use most frequently at work. There's some groundwork laid to support other dialects, but it's just the rudiments right now. Considering some of the differences are significant, I'm not sure how much code I'll be able to share between dialects.

## What It Contains

The repo currently contains a tokenizer, recursive descent parser for a subset of T-SQL (most SELECT, INSERT, UPDATE, DELETE and MERGE statement usages, IF/ELSE/END, BEGIN/END blocks and DECLARE/SET for variables), an abstract syntax tree (AST) implementation, symbol tables, and an AST visitor framework (with rudimentary stringifier, optimizer and validation visitors).

The unit test suite has pretty decent coverage, almost every syntax test does tree matching to make sure the parsed result matches what we expect, tests that the resulting tree is valid, and also tests round-trip parsing. That is, we parse the input, stringify the resulting tree, parse this second SQL string and make sure it produces an equivalent AST to the first one. In this way, every single unit test for the parser also tests our stringifier and validator logic, to make sure these things never fall too far behind.

The symbol tables are very rudimentary and basically serve to record the usages of variables and object identifiers. A post-parse visitor builds symbol tables from the parse tree. Because objects like tables, views, functions and procs are part of the "environment" of the database and those symbols aren't explicitly defined in any arbitrary string of SQL, this information would all need to be populated into the symbol tables ahead of time in order to really verify that we were only referencing symbols which exist.

The validator is very rudimentary with a lot of TODO notes.

The optimizer mostly focuses on expressions, reducing constant expressions to their results ("5 + 6" becomes "11", "'this' + 'string'" becomes "'thisstring'", and "CAST(11 as varchar)" becomes "'11'", etc). Because of the nature of SQL, it can be hard to do much optimization beyond that, because changes to query structure may change the result set.

## What It Does Not Contain

Besides the few things I described above as being incomplete, the project does not currently include any storage engine or runtime engine to execute statements. It is purely a parser and some utilities to operate on the constructed parse tree. Everything else is an exercise for a downstream project to tackle.

That said, there are a couple things that I would like to add to this library:

1. Parsing more statements. I don't think I'm going to get into DDL statements such as CREATE TABLE or CREATE STORED PROC (yet), but I would like to include things such as BEGIN TRANSACTION/ROLLBACK/COMMIT. I don't want to get into some structures which are very implementation-specific.
1. More validations. There are some issues we will only be able to detect at runtime, but there are plenty of things that we can try to catch immediately after Parsing to help validate the basics before we pass the AST to a downstream compilation or execution mechanism
1. More robust symbol table implementation, including availability of a "global" table for environmental objects, and better tracking of data types
1. Support for alternate dialects, especially Postgres SQL, SQLite and MySql, in that order. These dialects may prove different enough that they need separate everything from AST to validators, so if things get that complicated I'm probably not going to want to reproduce the whole project from the ground-up.

## Downstream Uses

This package is still in a very early state and I'm not making any big plans for it. There are a few little ideas I've kicked around though that might be worth bringing this project up to a more mature state:

1. Using SQL strings to query other data sources such as:
    1. Unstructured files
    1. CSV files
    1. `IEnumerable<T>`
    1. Json and XML documents
1. Doing some pre-execute validation and optimization of queries before sending them off to the server, so we can get superior error messages.
1. Tools to look at queries and recommend things like indices to help improve performance

The first bullet point might be the most interesting of the lot, because it gives us an opportunity to run tests against a dummy database, translating your production SQL queries into operations on test data instead of a live DB connection. It's interesting to think about, but if the parser and expression compiler aren't solid as a rock, they will just introduce false negatives into your test suite.

## Future Development

I don't have a lot of huge goals in mind for this project, it's just a fun little toy at this point. If other people found it helpful, I might consider focusing more attention on it. Right now it's just a way for me to better learn some parsing techniques and SQL as a whole. If you're interested [check out the Github repository](https://github.com/Whiteknight/SqlParser) and let me know what you think.