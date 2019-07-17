---
layout: post
categories: [JobHunt]
title: Architect Interview, Service Question
---

This is a continuation of [my previous post](/2019/07/10/archinterview.html) where I discussed some parts of our interview process. This time I'm going to go in-depth about my favorite part of the interview, which I think is the most insightful and interesting.

## The Question

The question, simple enough, goes something like this (you can adapt it to your needs, though I wouldn't give a question like this to a candidate who isn't expected to be an integral part of your design processes):

> We are sending email marketing campaigns through a third party email provider. The provider sends us notifications for Delivered, Opened, Bounced, SpamBlock and Click-Through events. We need to collect these events into a data store and make the data available for queries which are all well-defined and known beforehand. The data should have a retention period of 1 year. The system should handle 100 campaigns per day, with an average of 100 emails per campaign.

This is an open-ended question by intention. There is a lot of opportunity for the candidate to ask questions like "how are we receiving the events from the third party provider?" ("REST" or "HTTP API", etc) or "What's the timeframe for generating reports?" ("near real-time" or "within a few minutes"). We tend to start simple and then expand the requirements by, for example, increasing the volume of incoming messages. We can usually identify the weaknesses of the candidate pretty quickly, so I like to really dig in to the areas of strength, and see just how deep the knowledge goes. Nobody is an expert in everything, so find what they are an expert in and bring that part out to shine.

What I like about this question is that it is so open-ended and it gives you a chance to see how a person approaches a design problem and where they focus most of their attention.

Also, it's worth mentioning that we actually did have a similar requirement, did design such a solution ourselves and are currently running it in production. We had enough first-hand experience to be able to intelligently dissect the solutions which candidates provide and compare their solution to our own. I definitely recommend that, if you ask such a question of candidates yourself, that you ask something which aligns closely with a problem you've already solved yourself.

## Some Domain Insight

First it's worth recognizing that "email marketing campaigns" is basically code for spam. Even if the campaigns are opt-in, we still have to fight with some of the issues created by unsolicited spam: spam filters, reputation scores, etc. Every email sent will generate one "initial" event, which can be one of either "Delivered", "Hard Bounce" (the email address is invalid), "Soft Bounce" (the email inbox is full, but the address is otherwise fine) or "Spam filtered". So, for 10,000 emails sent, that's 10,000 guaranteed events we're going to receive. Of emails delivered for a marketing campaign, a 2% or 3% open rate is considered "pretty good", so that's another 200-300 events coming into our system, and when opened, even fewer people will click through your links (though, some people may click multiple links). Rounding up, let's say we're dealing with around 10,500 events per day. This number could be significantly higher in some situations, but not an order-of-magnitude higher.

Each event will probably consist of about 4 pieces of information: The campaign ID, the recipient email address, the name of the event and, maybe, a timestamp. If we assume about 300-500 bytes per record times 10,500 events, we have about 3-5Mb per day, or about 1-2Gb over the course of a year in the simple case. Less if we normalize and de-duplicate the data, more if we aggressively index it. Again, we're within an order-of-magnitude here, so less than 10Gb per year in all cases.

There's also some differences for how we may want to handle these different events. High percentages of Spam events indicate that we have a reputation problem to remediate. Bounce events indicate that the address is invalid or inactive, and needs to be removed from our roster. Click-through events mean that the recipient is particularly engaged in the campaign and may require some follow-up attention from the sales team.

The queries that we want to pull out the other side will likely be statistical: What percentage of each campaign are Delivered, Bounced or spam-blocked? What percentage are opened and clicked-through? All these reports will almost certainly want to be aggregated by campaign ID or rolled up to the entity (User/Company/Tenant) who triggered them. In other words, a user will log in and ask the system "show me relevant statistics for my campaigns".

None of these bits of information are strictly required for the problem. Some candidates bring some of this knowledge with them, sometimes they will ask, sometimes they will go through a requirements-gathering exercise to figure them out, and sometimes they'll ignore these details entirely and jump right into design. None of these approaches are disqualifying, but it is a point of interest to see how the person works.

Now, I'll get into some of the answers we receive and what they mean.

## The Simple Answer

The first, obvious answer is the simple one: A single "monolithic" application which receives HTTP webhook calls from the external provider, does the necessary processing, dumps the data into a simple data store (SQL or whatever is available would be fine for this) and then provides endpoints to fetch the pre-defined reports. It is simple, quick to develop, uncomplex and perfectly fits the stated requirements.

Most developers skip right over this simple answer and immediately move to something more complicated and "enterprisey". This is a little bit of a red flag, because the bigger solutions are more complicated and **complication must be justified**. Sometimes candidates are just showing us what they think we want to hear, and sometimes they're just looking to showcase their skills, or maybe they are anticipating the direction we're going and just preparing for the the eventual "big-data" conclusion ahead of time. It's not disqualifying to skip the simple solution, just a little annoying.

Some developers get this solution, but can't grow it to meet more strenuous requirements. This is a huge red flag, because it tells us that the person isn't ready for design of larger-scale systems, which is what we really need.

## The Common Answer, Overview

We say that we're increasing the volume of messages, so the simple solution gets erased from the board and we move on to the more common distributed version. The common distributed verison from people almost always contains the following parts:

1. An endpoint application to injest messages from the third party email provider, such as a REST-based HTTP API. This application may do some basic validation and transformation, and then will dump the message onto a queue or message bus.
1. Worker applications which read messages from the queue and dump the information into a data store, possibly doing validation/transformation if it wasn't done earlier.
1. A central database which holds all the received event information
1. A second service which queries the data store and provides client apps access to the predefined reports.

Just about every candidate gets to this point, with more or less level of detail. If they really struggle to get an answer like this together, or if they really can't (or won't) provide much detail, the interview is basically over and we'll maybe probe a few things and then move on to the end of the interview.

Otherwise, we've gained some insights into the candidate:

1. How comfortable are they on a whiteboard in front of people?
1. How confident are they about their ability to solve this problem, and the solution they've come up with?
1. How intelligible are the block diagram skills (We don't expect perfect UML, but we need to see that they've effectively communicated their design).
1. Did they take notes about the problem, or are they trying to just keep it all in their head?
1. Did they forget or choose to ignore any of the requirements?
1. Do they mention, and at what depth, some of the other major concerns like security, logging, reliability, performance, operational cost, etc?
1. Do they mention anything like patterns or principles for messaging, data storage or overall architecture?
1. Do they mention or show any understanding of Conway's Law, or ask about the organization of the existing teams?
1. Does the design explicitly include all the requirements, like the data retention policy?
1. Where do they start their design and where do they focus their attention?

That last part in particular is the most interesting to me. I've seen candidates who spend a lot of time on the database side: drawing entity relationships and table diagrams, estimating storage requirements, comparing and contrasting different storage engines, etc. Sometimes these database people will even move pretty quickly to a hybrid solution, one "primary" data store for the raw append-only event stream and another "secondary" data store for the aggregations and queries. Other people focus a lot on the messaging component, having a lot to say about how the messages move (pub/sub vs request/response, async fire-and-forget vs synchronous blocking, etc) and will have opinions about which messaging tool to use for which part. Other people will focus more on the application architecture of the individual parts, how the code is structured and organized. Some people will even focus on other things, such as where security boundaries are, what is the physical infrastructure like, etc.

It's very rare that a candidate can speak with equal depth and detail about all these parts. You often end up with lots of detail on one part, and very little detail on another. The data guys, for example, will maybe have two or three blocks connected by simple lines to represent everything that isn't the database. The messaging guys will often put up a beautiful pipes-and-filters diagram for the first half, but they hand-wave away the data store and the reporting components with a single drawing of a cylinder at the bottom labeled "data". Don't be disheartened, we're hiring people for their strengths, after all!

### Red Flag: Technology Evangelism

Beware the group of people who focus too heavily on one piece of technology, such as a particular database, particular message broker or particular hosting technology. Be wary of people who jump to a particular "silver bullet" technology to solve all their problems. "My preferred technology can easily handle all of this". Ask them to defend the choice: Why did you pick this technology? Why not one of the commmon alternatives? If the answer is "well, this is the only thing I know" that's a red flag by itself. It's not wrong to have a favorite tool, but it is wrong to not be able to technically justify the choice.

Also, please don't turn our interview into a sale-pitch or love-letter to a particular tool. I don't have the time for that kind of crap.

## Moving Up To Scale

No matter what solution they come up with first, we'll start dialing up the requirements to find the place where the first design breaks and needs to be reconsidered. If the design can handle a million messages per day, we'll crank it up to a billion. What do we do in the face of an unreliable VM host? What about network partitions? Basically, we want to see what kind of techniques the candidate is familiar with. We would like to see some of these kinds of answers expressed:

1. Using a load balancer up front to split incoming event load across multiple injest servers
1. Using queues, if they haven't already, for reliability and load distribution purposes.
1. Partitioning the queue by ID or topic to avoid overloading the queue infrastructure (especially since the nature of this data doesn't require strict ordering.)
1. Scaling out queue consumers across multiple servers to handle the processing work
1. Using a hybrid data strategy with a primary master data store for injest and a secondary store optimized for queries
1. Pre-aggregating or snap-shotting report data, in memory or in a fast cache (with some kind of replay if the cache is lost at any point)
1. Using some kind of async token-based request mechanism to request reports to be generated, instead of a blocking synchronous API call to generate it on demand
1. Partitioning the data up onto multiple data stores and using a map-reduce mechanism for queries

This isn't an exhaustive list of possibilities, just a smattering of common ones we've heard. If a candidate can't come up with anything like this, or something else to help improve the system, that's a serious red flag.

## Example Solutions

Here I'll present a few examples of systems which meet the requirements and are scalable well beyond 10k messages per day possibly with other benefits as well. In these solutions, I'll talk about some specific pieces of software, technology or infrastructure, and try to justify those decisions in the specific design. Notice that these aren't the only possible "correct" solutions, just a few of my own creation to serve as examples.

### Kafka Solution

1. A load-balanced injest service which does rudimentary validation, maps the data to a message format and dumps the messages into Kafka by event-type
	1. Kafka has configured retention of 1 year.
1. Reader processes which pull the messages out of Kafka and pre-aggregate data into a fast in-memory store like Redis or Memcached
1. A query API which provides access to the pre-aggregated queries via an internal REST api, and can trigger the readers to re-process the data if the caches are dead.

This is a "messaging" kind of solution where we are using Kafka, though other similar solutions could be used as well (a "streaming database", though this is a relatively new technology with few options, or a combination of a time-series DB and a traditional queue like RabbitMQ, etc) as our primary data store. Because Kafka streams data, we'll want some secondary store to hold the aggregated data somewhere fast. An in-memory cache (Redis, Memcached) would be perfect for this purpose. If the cache doesn't have the data we need (such as being out-of-date or having been wiped due to power cycling), we can signal the Reader processes to rewind/replay the stream from Kafka to get caught up again.

Two problems with this design are that Kafka becomes a single point of failure because of all the roles it plays, and that we have multiple applications accessing the report cache directly. Ask if the candidate can address these issues.

### SQL Server Data Mart Solution

1. A load-balanced injest service dumps messages into a local SQL database or other data holding area.
1. A DTS or ETL system (like SSIS) reads data from the local databases, validates, transforms, and dumps the results into a central SQL Data Mart
1. The central Data Mart uses a Star-Schema setup with append-only fact tables, partitioned by date
	1. A scheduled database job truncates partitions older than 1 year
1. Scheduled jobs can pull denormalized views of data out into a queryable BI or analytics system for reporting and querying

This is a "database" solution to the problem, with database technologies being utilized. Installing and managing Data Mart DBs and BI/Analytics platforms when they do not previously exist can be quite expensive, but if these features already exist in your network it can be relatively easy to expand them to cover this use-case as well. Make sure the candidate asks "do you have these things?" before going down this path, or else they are likely to be dismissive of cost-concerns throughout and will recomend million-dollar solutions to hundred-dollar problems. Also, you'll want to probe them about the long-term maintainability of these systems, how do we synchronize schemas between the various parts, do we use source-control for our DB object definitions, etc.

A few problems with this design are the cost considerations, the potential lack of sufficient DBA resources on the team to maintain it, and no obvious hooks for applications to act on the events in a timely manner.

### Distributed NoSQL Solution

1. A distributed NoSQL database such as Cassandra, Riak, CouchDB or ElasticSearch
	1. A separate timer-based application can be used to enforce retention policy, if the specific data store doesn't offer it already
1. An injest service dumps message data directly into the distributed data store
1. A query service uses map-reduce (either built-in to the data store or built separately) to pull queries

Distributed data stores like Casssandra, Riak, CouchDb, or ElasticSearch could all serve the central role here. A store with multiple write nodes would be able to handle high injest rates without needing a queue to help with backpressure. The distributed nature and map-reduce ideas can help keep queries fast and give some reliability guarantees. Notice that this solution will lean very heavily on the specific data store technology used, so make sure to probe about the fitness of that solution and expect the candidate to speak in-depth about it: Why was one chosen over the possibile alternatives? What are the strengths and weaknesses of the platform? Unlike the "Database Solution" above which has cost concerns, this solution often relies on open-source offerings or things which may be much more cost effective than SQL Server or Oracle DB, though not entirely without cost especially when hosting and maintenance questions arise.

Some problems here are the single-point of failure and situations where the injestion rate of the data store can't keep up during bursty periods. A queue might still be warranted here to help with reliability in these situations.

### Azure Serverless Cloud Solution

This solution is "serverless" and "cloud" using using Microsoft Azure offerings:

1. A serverless "Function" receives incoming messages and dumps them onto an Event Hub
1. A Service Fabric Service or hosted Docker containerized app reads the message off the Event Hub and dumps them into an Azure Data Warehouse
	1. A serverless Function on a timer removes data from the Data Warehouse older than 1 year
1. A series of serverless Functions serve queries in response to HTTP requests

A similar solution exists for AWS with Lambdas and Redshift. The beauty of the cloud solution is that the platform typically handles (or provides programmatic access to) scale up and scale down resources in response to demand. Expect the candidate to be able to talk about cost and cost-cutting strategies ("dynamic scaling", etc) when pitching a cloud-based solution.

### Some Other Ideas

1. If we aren't worried about a hard real-time or even near real-time query solution, using Hadoop as the central component is possible. Events are dumped into Hadoop and snapshot report processes are run periodically to aggregate the data. Expect the candidate to justify the delays in reporting.
1. A lot of the solutions above ignore the hosting of the various "apps" and "services". These could all live in some kind of hosting system like Kubernetes, Nomad or Apache Storm. Expect the candidate to explain the pros and cons of these kinds of approaches (added complexity, enforcing some design/development/deployment patterns that the team might not be familiar with, etc)

## Overview

This, I think, is probably the single most valuable question in an interview for architect-level positions (at least, for the kinds of Architects we're searching for) and the broad range of diverse responses I get tend to be quite informative. Expect a good candidate to be able to give an answer which is "good enough" (because of the time constraints and high pressure of the interview, we typically can't expect perfection) and be able to confidently defend their decisions *but* still be open to changes or suggestions. You want confident and adaptive, but not cocky or intransigent.




