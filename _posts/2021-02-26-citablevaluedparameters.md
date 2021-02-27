---
layout: post
categories: [projects]
title: Table-Valued Parameters in CastIron
---

I participated in an interesting little [conversation on Reddit about SQL and ORMs](https://www.reddit.com/r/dotnet/comments/lqjw33/writing_sql_in_c_or_when_you_should_not_use_orm/). I've discussed my [opinions on ORMs vs micro-ORMs previously](http://whiteknight.github.io/2018/07/23/ormproblems.html) and I won't get all the way into that. I realize my opinions on the matter are not necessarily the most popular or the most common. One comment on Reddit caught my attention:

> SQL has a power, but it is not ready for query decomposition. And you copy paste code, concatenates strings. LINQ is great technology which can avoid such problems...

This is really the weakness of the micro-ORM approach: You're dealing with SQL as low-level strings, and those strings are cobbled together using string concatenations and `StringBuilder`. There's also a really small barrier protecting you from injection attacks. You have to remember to properly escape and quote your strings before inserting them into the query. 

One area where CastIron has really struggled is in dealing with lots of input data to queries. Let's say I wanted to `INSERT` 100 rows into a table. I could do a bunch of this:

```csharp
foreach (var row in rows)
    sb.AppendLine($"INSERT INTO dbo.MyTable (ColA, ColB) VALUES ('{row.ColA}', {row.ColB})");
```

But that's kind of crappy, plus there's the obvious SQL injection opportunity with the `ColA` string value. So instead we can do something like this:

```csharp
for (int i = 0; i < rows.Length; i++)
{
    interaction.AddParameterWithValue($"colA{i}", rows[i].ColA);
    interaction.AddParameterWithValue($"colB{i}", rows[i].ColB);
    sb.AppendLine($"INSERT INTO dbo.MyTable (ColA, ColB) VALUES (@colA{i}, @colB{i})");
}
```

This is a little safer, but still not great, especially if you forget an `{i}` somewhere and then are referencing the wrong variable. You're not getting the SQL injection attacks, but it's still rickety.

Today I released v2.1.0 of CastIron to add in [table-valued parameters for SQL Server](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/sql/table-valued-parameters#passing-a-table-valued-parameter-to-a-parameterized-sql-statement). PostgreSQL supports a similar idea with lists, but it's going to be more work and is going to require different code and method signatures, so that one is coming later. 

To pass table values to an SQL query or stored proc, first you need to create a `TYPE` on the server. Then you can pass a `DataTable` as a parameter to the `SqlCommand`, referencing the name of the `TYPE`. See the link above for some examples. Now you can do this in CastIron, this example pulled almost verbatim from the test suite. Pay particular attention to the `.AddTableValuedParameter()` method towards the middle of the example, and the calls for `batch.Add()` towards the end of the sample:

```csharp
public class MyTestType
{
    public MyTestType(int first, int second)
        => (First, Second) = (first, second);

    public int First { get; set; }
    public int Second { get; set; }
}

public class TabledValuesQuery : ISqlQuery<int[]>
{
    private readonly TableValues[] _values;

    public TabledValuesQuery(TableValues[] values)
    {
        _values = values;
    }

    public bool SetupCommand(IDataInteraction interaction)
    {
        interaction
            .AddTableValuedParameter("tableValuedParameter", "MyTestType", _values)
            .ExecuteText("SELECT First + Second FROM @tableValuedParameter AS tvp");
        return true;
    }

    public int[] Read(IDataResults result)
        => result.AsEnumerable<int>().ToArray();
}

[Test]
public void TableValuedParameterFromObjectArray()
{
    var runner = RunnerFactory.Create();
    var batch = runner.CreateBatch();
    batch.Add("CREATE TYPE MyTestType AS TABLE ( First INT, Second INT );");
    var result = batch.Add(new TabledValuesQuery(new[] {
        new MyTestType(10, 5),
        new MyTestType(20, 7),
        new MyTestType(30, 9)
    }));
    batch.Add("DROP TYPE MyTestType;");
    runner.Execute(batch);
    result.GetValue().Should().ContainInOrder(15, 27, 39);
}
```

The `batch.Add()` calls allow us to batch up multiple queries together on a single connection. We use this to first create the `TYPE`, then to execute a parameterized query with a table variable, and finally to `DROP TYPE` when we are done with it. Unfortunately I don't think there's a way to create the type temporarily or contextually, so in production settings you're probably going to need to coordinate with the DBAs to make sure the types you need are created ahead of time. 

I'm working to add the equivalent list feature to the Postgres bindings, though it's going to look a little bit different. I'm also going to start working on some new extension methods which are going to significantly shorten the above example code into something much more manageable. Those and a few other small features and fixes will probably form a 2.2.0 release in the near future.