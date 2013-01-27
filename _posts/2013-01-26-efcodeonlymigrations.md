---
layout: post
categories: [DotNet, EntityFramework]
title: Entity Framework Code Only Migrations
---

I've been using Entity Framework 4.4 at work a lot recently, and as part of that I've been running
into some questions about how to do this or that with some of the new features, particularly code
first. Sometimes I'm able to find the answers I need from the Googles, but sometimes I've got to
sit down with VisualStudio and find the answers through good old-fashioned trial and error. Then,
I figure what's the point of having a tech-related blog in the first place if you can't share the
things you've learned there. I'll be sharing bits of what I'm learning as I go.

## Code-Only Migrations

The new Entity Framework releases have a feature called code-first, where you can write pure csharp or
VB code objects ("Plain Old Code Objects", or POCO), and have the Entity engine automatically
discern from those classes the shape of your DB tables and generate a change script to create them.
Most tutorials on the topic explain the process through the use of the Package Manager Console in
Visual Studio. I have slightly different requirements and so I'm going to try to do the same exact
process using the csharp APIs directly.

Here's a short but helpful blog post where I started my search:

[http://romiller.com/2012/02/09/running-scripting-migrations-from-code/]

### Create Your DbContext and POCO Classes

I won't go into detail about that here. There are plenty of cool resources for this purpose
elsewhere. For the purposes of the rest of this post, I'll assume you've got a `DbContext` subclass
called "MyDbContext". Even though you may not like to have it, your DbContext subclass must provide
a parameterless constructor to work with the Package Manager Console tools.

### Create a Configuration

A Migration Configuration is a class that derives from
`System.Data.Entity.Migrations.DbMigrationsConfiguration`. You can create one of these automatically
through the Package Manager Console with the `Enable-Migrations` command, or you can just create it
in code yourself:

{% highlight csharp %}
namespace MyProgram.Migrations
{
    using System;
    using System.Data.Entity;
    using System.Data.Entity.Migrations;
    using System.Linq;
    using System.Reflection;
    using MyProgram;

    internal sealed class MyConfiguration : DbMigrationsConfiguration<MyProgram.MyDbContext>
    {
        public Configuration()
        {
            AutomaticMigrationsEnabled = false;

            // These things are not strictly necessary, but are helpful when the assembly where
            // the migrations stuff lives is different from the assembly where the DbContext
            // lives. For instance, you may not want to run migrations from a separate
            // development-time console program, and not have that code included in production
            // assemblies.
            MigrationsAssembly = Assembly.GetExecutingAssembly();
            MigrationsNamespace = "MyProgram.Migrations";

        }

        protected override void Seed(MyProgram.MyDbContext context)
        {
            // TODO: Initialize seed data here
        }
    }
}
{% endhighlight %}


### Create a Migration

Next step is to create a migration. A migration is any class which derives from
`System.Data.Entity.Migrations.DbMigration`. You can create one of these manually, but it's much
easier to create them through the Package Manager Console with the `Add-Migration` command.

    Add-Migration MyMigration

Or, if you need some more options (if your solution has multiple projects, etc):

    Add-Migration -Name MyMigration -ProjectName MyProject -ConfigurationTypeName MyProject.Migrations.MyConfiguration

You may also need to specify `-StartupProjectName`, if your migrations live in a library assembly.

Als, you can specify a separate connection string from what is provided by the default parameterless
constructor of your DbContext by specifying  `-ConnectionStringName` (for a named connection string
in your app.config/web.config file) or `-ConnectionString` and `-ConnectionProviderName` to use a
value which is not in your app.config/web.config file.

What do all these options mean? Let's consider a solution with two projects:

    MyProgram.sln
        - MyProgram      (a .exe which references MyProgram.Core.dll)
        - MyProgram.Core (a .dll Class Library)


The project `MyProgram.Core.dll` contains our `DbContext` instance and the `MyProgram` assembly
has the app.config with connection string information.

If we want our migrations to live in `MyProgram.Core` we can use this command as our base (plus any
other options we need to add):

    Add-Migration MyMigration -ProjectName MyProgram.Core -StartupProjectName MyProgram ...

If, on the other hand, we want the migrations to live in `MyProgram`, the .exe instead of the .dll,
we can use this version:

    Add-Migration MyMigration -ProjectName MyProgram -StartupProjectName MyProject ...


If you do not specify `-ProjectName` or `-StartupProjectName`, the `Add-Migration` command will
attempt to use whichever project you have flagged as the "default startup project" in the solution
explorer (whichever project runs when you press F5).

What if I want to separate my migrations out into a different assembly entirely, one which isn't
included in my production deployment? Here's another example solution:

    MyProgram.sln
        - MyProgram                 (the production deployed .exe)
        - MyProgram.Core            (where our DbContext and model classes live)
        - MyProgram.DbMigration     (where our migration code will live, references MyProgram.Core.dll)


In this case, we can use a command like this:

    Add-Migration MyMigration -ProjectName MyProgram.DbMigration -StartupProjectName MyProgram ...

You're going to have to play with some of the options for different configurations. If the
`Add-Migrations` command says something's wrong, try tweaking your values and adding more info to
the commandline.

### Run the Migrations (Some Recipes)

Now that you've got migrations and a configuration, you can run the migrations manually. Here are
some snippets from a console program which does exactly this:

{% highlight csharp %}
private void DoDbUpdate()
{
    DbMigrator migrator = new DbMigrator(new MyConfiguration());
    migrator.Update();
}
{% endhighlight %}

Let's take a minute to step back and ask how this all works. You build your assembly and run it. The
`DbMigrator` class uses reflection to read out all classes from your assembly, and find the ones
which are subclasses of `DbMigration`. Each DB migration has a name, which is a combination of a
timestamp and the name you gave it in the `Add-Migration` command. In the database, there's a table
(or will be, after you run your first migration) called `dbo.__MigrationHistory` (it may be under
the "System Tables" folder). That table holds information about migrations you have already ran.
When you call `DbMigrator.Update()`, it searches for all migrations, removes the ones which already
have entries in the table, and orders them according to timestamp. This is the list of pending
migrations. You can get that list like this:

{% highlight csharp %}
private void DoDbUpdate()
{
    DbMigrator migrator = new DbMigrator(new MyConfiguration());
    foreach (string migration in migrator.GetPendingMigrations()
        Console.WriteLine(migration);
    migrator.Update();
}
{% endhighlight %}

You can also get the raw SQL script which is going to be used:

{% highlight csharp %}
private void GetDbUpdateScript()
{
    DbMigrator migrator = new DbMigrator(new MyConfiguration());
    MigratorScriptingDecorator scripter = new MigratorScriptingDecorator(migrator);
    string script = scripter.ScriptUpdate(null, null);
    Console.WriteLine(script);
}
{% endhighlight %}

Running the scripting decorator clears out the list of pending migrations from the migrator. If you
want to generate the script first (for logging) and then run the migration, you need to create two
migrators:

{% highlight csharp %}
private void GetDbUpdateScriptAndUpdate()
{
    MyConfiguration myConfig = new MyConfiguration();
    DbMigrator migrator = new DbMigrator(myConfig);
    MigratorScriptingDecorator scripter = new MigratorScriptingDecorator(migrator);
    string script = scripter.ScriptUpdate(null, null);
    Console.WriteLine(script);

    migrator = new DbMigrator(myConfig);
    migrator.Update();
}
{% endhighlight %}


Another thing we could try is to create a logging object, and use a logging decorator to log
progress. This mechanism will also output the raw SQL text, but will do so piecewise intermixed with
other information (so you'll need to filter out what is and what is not part of the SQL script):

{% highlight csharp %}
public class MyLogger : System.Data.Entity.Migrations.Infrastructure.MigrationsLogger
{
    public override void Info(string message)
    {
        // Short status messages come here
    }

    public override void Verbose(string message)
    {
        // The SQL text and other info comes here
    }

    public override void Warning(string message)
    {
        // Warnings and other bad messages come here
    }
}
{% endhighlight %}


Once we have a logger, we can use it in our migration:

{% highlight csharp %}
private void DoDbUpdateWithLogging()
{
    DbMigrator migrator = new DbMigrator(new MyConfiguration());
    MigratorLoggingDecorator logger = new MigratorLoggingDecorator(migrator, new MyLogger());
    logger.Update();
}
{% endhighlight %}

We can update to a specific migration, or we can rollback to a specific migration by name. Remember,
the "name" used by the migrator is a combination of the timestamp and the name you gave it at the
console.

{% highlight csharp %}
private void UpdateOrRollbackTo(string name)
{
    DbMigrator migrator = new DbMigrator(new MyConfiguration());
    migrator.Update(name);
}
{% endhighlight %}

And what if you want to completely trash the DB, undo all migrations, delete everything, and start
over?

{% highlight csharp %}
private void CompletelyTrashDb()
{
    DbMigrator migrator = new DbMigrator(new MyConfiguration());
    migrator.Update("0");
}
{% endhighlight %}

## What's My Use Case?

So what exactly is my use-case here? Why don't I just stick with the Package Manager Console like
many other tutorials do? I have a few criteria:

1) I need to seed the new DB with a lot of complex data, pulled from another source, which needs to
   be updated regularly.
2) I may have more than one DB, for multiple instances of my application. Connection strings for all
   of these may be kept in another DB or a file or somewhere else. All of these need to be
   kept in sync, and a script that runs a migration on all targets is better than a command which
   runs on only one and needs to be manually updated.
3) I'd like the ability to log the SQL scripts which are used for the migration, for various
   purposes.
4) I'd like to be able to do some scripted unit testing where we create and migrate a test DB from
   scratch, seed it with test data, and use that for testing. I would like these temporary test
   DBs to be identical to the production ones.

Overall I think the new Entity Framework Code-First features are really cool, and remind me very
closely of the equivalent db migrations scripts in Rails, but we have a little bit more control
over it here because we can incorporate the DbMigration process into our application logic.
