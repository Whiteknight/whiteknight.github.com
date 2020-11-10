---
layout: post
categories: [projects]
title: CastIron v2.0 Released
---

I have released [CastIron](https://github.com/Whiteknight/CastIron) version 2.0 to [nuget](https://www.nuget.org/packages/CastIron.Sql/) today. It has been a long time coming with a lot of development work and a bunch of new features. Some of the new features I mentioned in a [previous post](http://whiteknight.github.io/2020/10/20/castiron2.html) about the 2.0 beta. Several new things have been added since that point as well.

## Rewritten `MapCompiler`

I won't go into as much detail as I did in the previous post, where I really described the rewrite and the motivations which lead to it. Suffice to say that the internals of the map compiler are completely rewritten, are much more powerful and flexible, and support many many more options and data structures.

## New Mappable Types

There are several new types that you can map to in the new compiler. `ValueTuple<>` (for .NET Standard2.0 builds and later) and `ISet<T>` are probably the biggest. Value Tuples in particular enable some really fun query opportunities:

```csharp
var values = results.AsEnumeable<(int id, string name)>("SELECT Id, Name FROM MyTable").ToList();
```

You don't even need to define a `class` to hold your data, you can just use a ValueTuple and get named properties for free.

## Type Configuration

Again, I've already covered the detail previously. In CastIron v2.0 you have a heck of a lot of control over how data is mapped from columns to objects, and how your objects are materialized. Here's a piece of example code which shows some of the power you have available:

```csharp
var output = results
    .AsEnumerable(s => s
        .For<TypeA>(a => a.UseConstructor(typeof(int)))
        .For<TypeB>(b => b.UseSubclass<TypeBSubclass>())
    )
    .FirstOrDefault();
```

## SQL Server Adaptor

SQL Server code is moved to a separate library `CastIron.SqlServer`. Now the default `CastIron.Sql` library is provider agnostic. To use CastIron you will need to reference both the `CastIron.Sql` library and the provider library of your choice (currently choose from `CastIron.SqlServer`, `CastIron.Postgres` or `CastIron.Sqlite`).

## Runner Configuration

`ISqlRunner` implementations are very small and easy to create, so now they will hold all settings related to queries and you won't have to set your timeouts, monitoring or transaction settings on every query. Simply call the setup options when you build the runner:

```csharp
var runner = RunnerFactory.Create(connectionString, build: config => config
    .SetTimeoutSeconds(20)
    .UseTransaction(IsolationLevel.Serializable)
);
```

If you need multiple runners with different sets of options, you can easily create more. They aren't expensive.

## Caching

Caching is redone in CastIron v2.0. In v1.0 the cache was keyed using a serialization of both the `IDbCommand` and the list of columns from the `IDataReader`. Building up a large string key was unweildy and it could only be successfully done in certain cases: You couldn't cache any query which contained any options which weren't serializable (like a factory method for creating an object instance, for example). In v2.0 we now cache queries using the `ISqlQuery` object instance as the key. If you reuse the same Query object over and over again, you can reuse the same compiled map function for it each time. If you want to use multiple separate query object instances, but they all share the same shape of the result set (same columns with the same data type and names, in the same order, mapping to the same type of object) you can supply your own custom key.

The cache instance is overrideable, of course, and it is part of the `ISqlRunner`. If you create a new `ISqlRunner` you will get a new cache instance. You can also set a cache instance when you create the runner, if you want to share a cache between runners or use a default global instance:

```csharp
var runner = RunnerFactory.Create(connectionString, mapCache: MapCache.GetDefaultInstance());
```

Caching is opt-in, to help avoid gotcha situations where a hidden cache grows and grows until your server has memory problems. In order to turn on caching, you have to call a method in the `Read` method of your query object:

```csharp
public MyObject Read(IDataResults data)
{
    data.CacheMappings(true);
    return data.AsEnumerable<MyObject>().Single();
}
```

You can also specify a custom key there, if you don't expect to reuse the same query object instance but you believe the query plan can be mapped:

```csharp
data.CacheMappings(true, "my-mapping");
```

## Enum Compilation

I sort of can't believe that we left enums out of the explicit list of mappings for v1.0, but now they are properly supported by v2.0. Here are two examples lifted directly from the test suite:

```csharp
private enum IntBasedEnum
{
    ValueA,
    ValueB,
    ValueC
}

[Test]
public void Map_CaseInsensitiveString()
{
    var runner = RunnerFactory.Create();
    var result = runner.Query<IntBasedEnum>("SELECT 'valueb'").Single();
    result.Should().Be(IntBasedEnum.ValueB);
}

[Test]
public void Map_int_IntBasedEnum()
{
    var runner = RunnerFactory.Create();
    var result = runner.Query<IntBasedEnum>("SELECT 1").Single();
    result.Should().Be(IntBasedEnum.ValueB);
}
```

It's a rarely-used feature in C# but you can use other number types as the basis for your enum. You can, for example, define an enum which uses `byte` as the underlying storage instead of the default `int`:

```csharp
public enum ByteBasedEnum : byte
{
    ValueA,
    ValueB
}
```

On the rare occasion when you want to use this feature, CastIron supports it and will map numbers to your enum type as expected.

## Other Bits

In addition to the big user-facing changes, I have been busy at work behind the scenes improving code quality, improving test coverage, and generally trying to make a better and more sturdy project. Overall I think v2.0.0 is a much better library than v1.0.0 was, and I'm looking forward to making similar improvements to it in the future as well.

## Next Steps

I'm putting CastIron on the shelf for now. I want to let the current feature set sit and marinate for a while, to see if I can get some feedback from other people and so I can employ it in other projects downstream. I don't have any major changes in mind right now that would kick-off a v3.0 development path, but there are always a few little nits here and there which might eventually turn into a v2.1.0 eventually. For now I'm pretty happy with this library as it currently is, so I'm going to work on something else for a while.
