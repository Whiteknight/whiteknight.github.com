---
layout: post
categories: [Patterns, EntityFramework]
title: Repository and EntityFramework
---

I've seen more than a couple of tutorials online talking about how to implement a Unit Of Work and
a Repository pattern using Entity Framework. What you will inevitably see is something like this:

{% highlight csharp %}
public class UnitOfWork {
    private readonly DbContext m_context;
    public UnitOfWork(DbContext context) { m_context = context; }

    public void SaveChanges() { m_context.SaveChanges(); }

    public Repository<T> GetRepository<T>() { return new Repository<T>(m_context); }
}

public class Repository<T> {
    private readonly DbSet<T> m_set;
    private readonly DbContext m_context;
    public Repository(DbContext context) {
        m_context = context;
        m_set = context.Set<T>();
    }

    public T Load(int id) { return m_set.Find(id); }
    public IEnumerable<T> LoadAll() { return m_set.ToList(); }
    public void Insert(T obj) { m_set.Add(obj); }
    public void Update(T obj) {
        m_context.Entry(entity).State = System.Data.Entity.EntityState.Modified;
    }
}
{% endhighlight %}

It's also worth mentioning that you could make a few changes and have a Repository which does not
rely on a Unit Of Work, if you don't need the kinds of transactional consistency that a UOW brings:

{% highlight csharp %}
public class Repository<T> {
    private readonly DbSet<T> m_set;
    private readonly DbContext m_context;
    public Repository(DbContext context) {
        m_context = context;
        m_set = context.Set<T>();
    }

    public T Load(int id) { return m_set.Find(id); }
    public IEnumerable<T> LoadAll() { return m_set.ToList(); }
    public void Insert(T obj) {
        m_set.Add(obj);
        m_context.SaveChanges();
    }
    public void Update(T obj) {
        m_context.Entry(entity).State = System.Data.Entity.EntityState.Modified;
        m_context.SaveChanges();
    }
}
{% endhighlight %}

In either case, this seems all well and good. We've implemented a Repository and possibly a Unit Of
Work that both use the underlying `DbContext` and abstract it away so we can mock out our data store
for unit tests. Win, right?

What should be immediately obvious here is that our code doesn't do anything. We haven't written
a Repository, we've written an **Adaptor**. EntityFramework already has a Repository for us. It
doesn't have the word "Repository" in the name, but it's a repository nonetheless.

In EntityFramework, `DbContext` *is a unit of work* (among other things) and `DbSet<T>` *is a
repository* (among other things). These patterns are already properly employed, so all you need to
write for your code is a thin adaptor to abstract away the DB details and be able to use a class
with the word "Repository" in the name.

When we are talking about classes like this which are very thin, it's worth asking if they are
required at all. That is, is it worth the modest amount of effort required to implement this adaptor
in the first place? Should we spend time wrapping up `DbContext` in a custom `IUnitOfWork` and
wrapping up our `DbContext` in a custom `IRepository<T>`?

Arguments could be made either way and, of course, it may just depend on the needs of your
application and your style of writing code. If you don't like the Respository pattern, you obviously
don't want to do this. But, for people on the fence, we can do a little bit of cost/benefit
analysis.

**Reasons Why**

1. We can easily mock our `IRepository<T>` and `IUnitOfWork` dependencies in our unit tests, giving
   us the ability to easily isolate tests from our database (making them faster and more reliable).
2. We provide a much smaller, simpler interface on the EntityFramework types which are, arguably,
   bloated. Consider this something of an extension of the Interface Segregation Principle (ISP).
3. It gives us isolation from EntityFramework, so we could potentially make changes to our ORM or
   our storage mechanism without having to make sweeping changes outside our data layer.
4. You can more easily include these adaptor types in other existing machinery which operates on
   Repositories over different stores.
5. We gain the ability to include our `IRepository<T>` in our domain layer, and inject the
   implementation elsewhere. This is in accordance with things like the Clean Architecture or the
   Onion architecture, which both bring their own benefits. (EntityFramework does a very poor job
   of providing its own interfaces for these purposes, so if you want the architecture to work, you
   need to provide some kind of adaptor and IRepository is as good as any).

**Reasons Why Not**

1. It does look a little like needless indirection, especially when EntityFramework is already
   implementing the patterns we need.
2. We can stub in `DbContext` and `DbSet<T>` in our unit tests to avoid hitting the database, giving
   us isolation (though, admittedly, doing this is much more difficult than employing a Mock Object
   framework like Moq).
3. The reality is that you aren't going to switch to a new ORM, especially if you're only employing
   the few features which can be cleanly exposed through the standard Repository interface.
   EntityFramework does those things no worse than any alternative, and the pain of switching ORM
   to get one with the same features implemented just as well isn't worth doing. (if you're
   employing more powerful, ORM-specific features, you probably aren't using IRepository because
   that abstraction boundary is so limited).

There's no magic bullet argument in either case, so it all comes down to preference. Is the
relatively small amount of code you need to write for this purpose worth the effort? You'll have to
make this decision for yourself.

Personally, my style of writing code almost always benefits from having these Repository adaptors
so I tend to provide them unless a simpler solution or a more flexible one is called for. For
example, if my only database interactions are some complicated queries which don't fit nicely into
the classic Repository interface, I may skip that but instead write other wrappers around
EntityFramework such as a Table Gateway. I would almost always want to have a wrapper of some sort,
in any case, because I tend to structure programs very much in an Onion or Clean style.
