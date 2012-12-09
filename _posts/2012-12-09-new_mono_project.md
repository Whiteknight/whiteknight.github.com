---
layout: post
categories: [Personal]
title: Working On a New Project
---

As they say on my son's favorite Thomas The Tank Engine, yesterday an idea flew
into my funnel. I was [doing some work on my bathroom](http://accidentallycooking.wordpress.com/2012/12/08/shower-problems/)
when I got an idea for a new website that I would like to make. I want to make
this site for a few reasons: First, I'm going to be using the new ASP.NET MVC
framework at [my new job](/2012/11/20/new_job.html) eventually, and I wanted to
practice with it. Second, I have been getting motivated to do more programming
in general, and a new project that I'm hot on seems like just the thing to get
me moving again. Third, my idea for this site is relatively straight-forward
but should offer some good practice and interesting technical challenges. Fourth
and finally, this website I'm thinking about is actually something that I would
like to use myself (even if nobody else joins me).

I still don't have a [new laptop](/2012/09/14/sept_status_update.html) yet,
though I've been shopping for one in earnest. I figure I'll have it shortly
after the holidays. In either case, for the time being I'm stuck with my current
crappy laptop, which exclusively runs (the decidedly not-crappy) Linux. If I'm
going to make a new ASP.NET MVC website on this box, it's going to have to use
[Mono](http://www.mono-project.com/Main_Page).

And that's fine. [I've used Mono before](/2010/11/03/blogger2jekyll.html) and I
like it plenty despite it's shortcomings.

I scratched out a few ideas and designs in my notebook. I decided I wanted to
use some kind of dependency injection/inversion of control/service locator
feature. I've used [Unity](http://unity.codeplex.com/) in the past and loved it,
but I wouldn't be against using [Ninject]() or something else too. I also want
to use some kind of ORM to make persistance a little easier.

Now, I can already hear some people mumbling to themselves about all the many
flaws of ORMs. I won't even bother to list them or link to the (many) pages
where they are discussed on the interwebs. Use your imagination. In any case,
I'm not detered and ORMs actually make good sense for the project I'm thinking
about, if I can find the right one.

I thought about using MongoDB, but after thinking hard about work flows and data
relationships in my site, I think a regular, SQL-based relational DB would just
be a better fit in this instance. I'd probably like to stick with MySQL or MS
SQL Server, initially. (This is not to say anything about the relative merits of
one type of DB over another, just that one type seems to be a more natural fit
for this particular problem domain and I'd like not to be shoehorning in the
wrong software for the wrong reasons. Don't get my involved in your holy war.)

The problem, I discovered, is finding a good ORM that's worth using, doesn't
introduce more hassle than it saves, and actually works (with examples) on Mono.
So far, my search is proving to be a little bit fruitless.

1. **[NHibernate](http://nhforge.org/)** seems like a common and popular choice,
   but the large amounts of required XML configuration make me sick to my
   stomach. I would far prefer something that I can do in pure C# code without
   large amounts of externa config.
2. **[Castle ActiveRecord](http://www.castleproject.org/projects/activerecord/)**
   builds an ActiveRecord-like interface on top of NHibernate. In theory you get
   all the power of NHibernate without the XML headaches. However, this package
   is listed on the castle website as being "Archived" and "no longer being
   worked on". Also, I can't find any real examples of using it on Mono. I'm
   not going to start a new project (which presumably could be active for years)
   by starting on an old and unmaintained foundation.
3. **[db4o](http://www.db4o.com/)** It looks to me like this little project uses
   it's own custom DB file format and doesn't connect to existing databases. I
   think I'd really like to stick with an existing DB, and not use something
   custom.
4. **[Linq-To-SQL](http://msdn.microsoft.com/en-us/library/bb425822.aspx)**,
   probably using the SQLMetal code generator, would seem like a decent option
   except it doesn't seem to be well-supported on Mono, and there's the issue of
   having to generate a whole bunch of pure-data objects, which will need to be
   laboriously mapped to and from my actual object type definitions. I've seen
   the kinds of morass that this kind of situation can lead to, and I'm not
   interested in going this route if it is even possible to traverse.
5. **[Simple.Data](https://github.com/markrendle/Simple.Data)** Is a newer
   option which uses all sorts of fancy modern C# features to provide an
   extremely flexible, extremely natural-looking interface for accessing a
   database. It's supposed to work on Mono, but I've not figured out a good way
   to get it (and a MySQL connector, and prerequisites) installed in a
   reasonable way for Mono. The docs suggest NuGet, but I can't get NuGet
   working on my box (and the [NuGet devs don't seem to care about Mono too
   much](http://nuget.codeplex.com/workitem/1271))

Overall, the experience of trying to get this project working on Mono has been
frustrating. I understand what a big engineering task Mono is in general, and
how much work goes into getting a diverse ecosystem of software to work together
nicely on a VM that's supposed to be cross-platform with low barriers to entry.
I get all that. Although, for the purposes of this project I really wanted to
start writing some code sooner than later and not have to fight with so much
infrastructural stuff. I suppose I have a few options: I can wait till I get a
new laptop and do things on a windows partition or VM instead. Or, I can keep
fighting with this setup to try and get things to work. Finally, I guess I could
port over my idea to Ruby on Rails, another platfrom that I'm interested in
learning more about.
