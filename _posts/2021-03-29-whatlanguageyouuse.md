---
layout: post
categories: [Philosophical]
title: The Language You Choose
---

I saw a nice blog post about programming languages, and more importantly about how it's [not important which language you use](https://georgestocker.com/2021/03/28/no-one-gives-a-shit-what-programming-language-you-use/).

> woodworkers take pride in the finished product, and seeing that finish product being used. They are not all hot on the tools they use, and nor do they value the art of planing for its own sake. No, it’s always in furtherance of the ultimate goal: to produce something another human will enjoy using.
> Or put another way, no one gives a shit what programming language you use.

I wouldn't say that "nobody gives a shit" *per se* but it really isn't the biggest driver of quality or anything. If you were choosing to use a language that was radically ill-suited for the task at hand, you might get a bit of interest in the method ("oh, it's fun that it's possible to do that!") but you wouldn't be considered to be following best practice. If you want to create a website in COBOL in your free time, have at it. That sounds like it could be a fun challenge. But if you're doing that at work, on the company dime, I would accuse you of gross malfeasance. 

The choice of what language to use is dependant on a lot of factors. I'll list a few:

1. What language(s) is your team comfortable using?
2. Or, what language(s) is your team willing and able to learn?
3. Do there exist sufficient learning resources, for free or for a reasonable price, to bring your team up to speed if they aren't already?
4. Will you be able to hire more team members in your geographical area and quickly ramp them up to work on your project in this language, or to replace people as members of your team leave? (Or, if you're able to work remotely, you don't have to worry about location)
5. Will you be able to afford competitive salaries for programmers in this language, in your area?
6. Is there sufficient support available if you run into problems? This could be paid support or free community support, in which case is there a sufficiently large and active community to provide free support?
7. Is there a sufficient ecosystem of third-party libraries available, so that you can do interesting things without having to reinvent everything from scratch?
8. Do there exist sufficient tools to test, build, deploy, monitor, debug, and address problems as they arise? Do there exist good editors, linters, analyzers and/or IDEs for your language and associated toolchain?
9. Will it be easy for new team members to read, understand, and modify work done in this language, if older team members leave?

If you're selecting a language because it is new, or because it has a cool feature, or because some of your developers want to add that keyword to their resumes, you're doing it for the wrong reason. If you come up with satisfactory answers to all the above questions, then it really doesn't matter what language you use.

Let me share one anecdote (which I may have shared before on this blog, but I can't be bothered to search for it right now). Organization where most code is written in C# is moving towards a microservice kind of infrastructure. One of the senior developers says, reminding everybody that they should be able to pick the best language for the job, suggests that the next service would be best written in Ruby. This developer writes the new service in Ruby, adds "Ruby" to his resume, and then promptly gets a new job somewhere else. Now the organization has a service written in Ruby, nobody on staff proficient enough in Ruby to maintain it, and eventually has to allocate resources to rewrite the service in C#.

I'm certainly not saying anything bad about Ruby. It's a fine language. I've used it earlier in my career to good effect, and might recommend it to some people. My point here isn't that Ruby is bad, it's that this organization's team couldn't support it, and it was a bad decision to introduce it in this case. A "good language" can still be the wrong decision in many cases.

Keep in mind that [code is a liability](https://medium.com/building-the-system/1970-01-01t00-00-00-00-00-8813a0ac62e5) not an asset. All software will eventually have bugs or deficiencies, and it will be work for your team to diagnose and fix them. The more proficient your team is at doing it, the lower the expense will be. If you aren't accounting for that expense, you aren't making a wise decision, you aren't doing your due diligence, and you aren't exercising responsible leadership. (This link actually seems to suggest a poly-lingual approach, which still seems like it creates more liabilities than it solves. Even if individual members of the team "move on", the core competencies of the team will not move as quickly, on average, as the individuals do. It's all about the team. It's important for management to make sure that the team continues to have the necessary competencies to maintain the old systems while also developing new competencies to create better systems in the future).

This conversation reminds me of an older article by Paul Graham about [his use of Lisp](http://www.paulgraham.com/avg.html). In that article, he talked about why he used Lisp and what he gained from it. It's a very interesting read. One of the arguments he made, which I'll admit is quite compelling, is this:

> A suspicious person might begin to wonder if there was some correlation here. A big chunk of our code was doing things that are very hard to do in other languages. The resulting software did things our competitors' software couldn't do. Maybe there was some kind of connection. I encourage you to follow that thread.

There are several reasons I don't use Lisp and why I wouldn't consider using it in the future. Part of the reason is this quote from the same post:

> there isn't room here to explain everything you'd need to know to understand what it meant. In Ansi Common Lisp I tried to move things along as fast as I could, and even so I didn't get to macros until page 160.

I'm skipping over a lot of context here, but the gist of it is that Lisp macros are a particularly powerful feature that other languages don't have and Paul's team uses these macros to great effect. Even if macros enable extremely cool things, and even if those things mean higher development velocity for Paul's team, it still takes 160 pages of briskly-paced text to even introduce them to a learner. You have to be unusually dedicated to read 160 pages of a programming book on any language, but to have to read 160 pages just to get to the point that you can begin to understand "20-25%" of Paul's source code seems like a hell of a learning curve to me. You can get moving in one of the C-family languages much more quickly than that, in my experience. 

It's cool that Lisp worked for Paul's team. It's really fun to read about it. There are lessons that I would like to learn from this and try to apply to my own work. But I wouldn't, in a million years, suggest to write a production-grade application in Lisp for any team that I've ever worked on. 

