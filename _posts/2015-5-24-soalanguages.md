---
layout: post
categories: [Architecture, Design]
title: SOA Language Selection
---

When you hear about Service Oriented Architecture in general, and Microservices specifically, you frequently hear about one of the common benefits: You can use the right tools for each individual job. Want to write a service with Node.js? Go for it. Want to write a service in Python or Ruby? Do it. Each individual service is so small, the thinking goes, that you can afford to experiment. Later, if you have to rewrite, it's easier to do.

Here's a story of a team which I was not a part of. They were a .NET shop with plenty of JS experience from frontend work. They had a job, to write a new service for doing authentication over Active Directory. The developer on the project didn't know how to do the job with C# or JS, but he did know of a Ruby Gem that solved the problem and exposed a friently API. He petitioned for and received permission to write the service in Ruby, with Rails.

A few months later, when that developer left for greener pastures (his resume now had "Ruby" listed prominently towards the top!) the team was stuck in a pickle: How do we maintain and even improve this service, which is written in a language that nobody else on the team knew? Nobody on the team knew Ruby, and certainly none of them knew Rails. In an enterprise-wide architecture with dozens of .NET-based services, this one little Ruby project stood out as a growing problem. Eventually, when the authentication system needed to be upgraded, this one little project needed to go through a complete rewrite. Nobody on the team was able to do the necessary upgrades in place.

When people are talking about using the right tool for the job, one thing that is almost always missing from the calculations is the availability of resources.  How often do we hear the following kinds of things?

"We'll deploy this service to a Linux server" when nobody on the Ops team has experience administering Linux.

"We'll use MongoDB for this application" when we have a dozen DBAs on our data team with experience in SQL Server, and none with experience in MongoDB.

I'm not saying we can't ever use unfamiliar technologies and that you can't, as a team, make a focused effort to develop new skills. What I am saying is that the availability of competent resources to develop, deploy, monitor and maintain the software needs to be taken into account. If there is a compelling technical reason to use something new, by all means go for it. But if you can't demonstrate the benefits of a new technology outweigh the core competency of your team to leverage existing technologies, maybe you shouldn't go down that road. Being different for the sake of difference (or worse, so your developers can pad their resume with new in-demand buzzwords) isn't a reason to use a foreign technology.

Yes, with a proper distributed architecture you *can* write different services in different languages with different backend technologies. I'm just saying that because you can do something doesn't mean you *should*. Make sure your team is able to support the project through its entire lifetime, keeping in mind that people will leave your team and need to be replaced, and all the other headaches of team management too.
