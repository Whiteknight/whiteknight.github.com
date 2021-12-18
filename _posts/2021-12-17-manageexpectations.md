---
layout: post
categories: [Philosophy]
title: Good Programmers Manage Expectations
---

Early in my career I thought that the way to succeed as a programmer was to be the most technical: Learn all about the languages, concepts, refactorings, patterns, styles, architectures and components. I studied and I practiced, and I tried to become as good at these things as I could be. This brought success, to a point. It turns out that there's a lot more to programming, especially at the higher levels.

My first clue that something was wrong was when I read [Domain Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) by Eric Evans. I expected that book to be all about the techniques, strategies and patterns for doing effective domain modeling. There was some of that, to be sure, but the major thrust of that book was all about communication between developers and non-developers. I wasn't looking to learn about *soft skills* but this is exactly what was being presented. Then I went back and looked at some other books and resources and saw a similar trend. [Design Patterns](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612/), as another example, isn't all about just having a toolbox full of cookie-cutter solutions to simple technical problems. The idea of patterns is in having a rich *language* which you can use to communicate ideas in a way that is succinct, precise and efficient. 

A developer's job, from a high perspective, is to take requirements described in human language and convert them into instructions for a computer to execute. In a real sense a developer is a *translator*, and translation is all about facilitating communication.

Communication with people is hard, but it's extremely necessary. Unfortunately we don't always work to improve our communication skills as much as we do our technical skills. People put "communication skills" under the requirements section of a job posting, or in the skills section of their resume, and just assume that the box is checked. This isn't enough. Communication and language are skills that you need to constantly work to practice and improve, and we as developers don't always do a great job of this.

The other day I got involved in a [discussion on Reddit about burnout](https://www.reddit.com/r/programming/comments/mm12pt/the_project_that_made_me_burnout/) which was about a very [interesting and impactful blog post](https://www.jesuisundev.com/en/the-project-that-made-me-burnout/). If you have a chance, I definitely encourage everybody (especially junior devs) to read that post. 

Burnout is tough. I've been there. I still have bouts of it from time to time. If you can't avoid it the best way to treat it that I've found is to get your hands off the keyboard for a while and let the natural enthusiasm and inspiration come back to you. You can't always do that in a professional setting. Sometimes you still have to grind out the daily work. But, you should find opportunities to decrease your intensity sometimes when you need it. A good boss will understand. Luckily, I think a lot of the time burnout is avoidable, but **communication is key**. Notice a trend here? The technical parts of your job are all about communication, and the non-technical parts of your job are also all about communication.

In this post I'm going to give a few pieces of advice to developers, especially junior developers, about how to use communication to be more comfortable and effective at your job. Some of these things are not at all obvious, and seem like they might make you unpopular, but a mature organization will understand the value and appreciate you more for them.

## Speak Truth to Power

The first thing you need to learn is that you have to **speak truth to power**. The quality of decisions that people make is related to the quality of the information they have available. Bad managers will make bad decisions regardless. There's nothing you can do about that. But good managers have a chance to do right if they have the best, most honest and up-to-date information.

First, don't say "I can do it" if you can't. If you create an expectation of success, you had better deliver or be prepared for the consequences. If you say you *will* do something you better do it. This is part of the truth that you should speak. If you can't do something, on the other hand, be prepared to say that too. You might not want to just come out and say "I can't do it". That does sound a little helpless and defeatist. However, you can communicate the same idea with language that is a little bit more measured and professional. Some lines you can try:

> This deadline is tight. Can we adjust the deadline or get some more resources assigned to it?

> This sprint is feeling a little bit over-stuffed. Can we adjust scope to give us a better chance at success?

> I'm worried about quality if we keep up this pace. We should alert QA that we are going to need extra testing this cycle.

And, when things start to look very dire:

> My confidence in delivering by the deadline is very low. We should start planning for contingencies.

> We aren't on track for delivering by the deadline. We should make sure items are prioritized properly so we can focus our effort on the most important ones.

If you tell management that you aren't going to deliver, and then you don't deliver, at least you were honest. At that point it's a management failure to either not take steps to ensure success or to not prepare for the fallout from failure.

If you think the project is heading towards failure, you need to bring it up. Repeatedly. Don't just mention it in a one-on-one meeting. Don't just mention it in an occasional email. Bring it up in every single standup. Make sure that everybody hears it, you're all in this together as a team, right? Make sure your manager hears you and understands you, and make sure there's a paper trail in case your manager is the kind of jerk who would throw you under the bus by claiming the failure to have been a surprise.

What I'm not suggesting is that you become a dour complainer in every meeting, always preaching doom and gloom and the end is nigh! Try to be optimistic where possible, and recognize that a little bit of determination and extra-ordinary effort can deliver things that might have seemed out of reach. But, when things are consistently not going well, and you are off-pace enough to make success unlikely, you have to make sure you give that information to the people who need to hear it.

## Offer Compromise

You've probably heard this adage before:

> Good, fast, or cheap. Pick two.

It's more than just saying you want this *or* that. All three of these things are knobs that can be adjusted in very fine-grained ways. And the "good" part of this adage accounts for more than just code quality, it encompasses every facet of the released product. Even if you have no (major) bugs and the maintainability of the code stays high, you may have to cut features from your plan or sacrifice on things like fine-tuned usability or optimization. This means there are even more knobs to be adjusted. All these knobs lead to a huge space of possibilities, solving a very dynamic equation in terms of many very dynamic variables. Decision makers can be overloaded into **decision paralysis**. There are often too many possibilities to consider. Don't just tell your manager that the deadline is going to be missed or that the project is going to fail. Try to offer some specific suggestions, alternatives and compromises to get the conversation started, and help the people who make decisions to start on their decision-making process.

Can we review some of the requirements and abandon some of the ones that are low-impact and high-effort? Or, in lieu of abandonment, can we reschedule some of the lower-priority features to future release? Can we simplify some of the requirements to reduce implementation effort? Can we maybe bring on an extra resource, at least a few hours each week? Can we (temporarily) cancel a few unnecessary meetings or decease meeting attendance so we can have more productive man-hours per week? Are there changes to the planning, approvals, testing or deployment processes that we could make to free up a couple man-hours each week? Can we recycle some existing work instead of creating new? Can we leverage a third-party component instead of building that functionality in-house? Can we work on some things like performance tuning later, assuming that initial adoption of the feature may be slow? Can the project be released in a beta mode, so we can continue to iron out bugs and get feedback without customers getting angry? Can we triage bugs and push off fixing the low-impact ones until later?

Try to offer specific possible solutions, even if it's not your job to ultimately decide on which ones are taken. Your manager may find it much easier to select between a few concrete options than to try and imagine a solution in the entire space of possibilities. Your manager may be able to pick one of your solutions as-is, or may use them as the basis for suggesting some kind of compromise of her own. Either way, you aren't just complaining, you're offering solutions and a good manager should appreciate that. Don't just throw grenades over the wall and run. Describe the problems, offer solutions, be part of an active discussion.

## Manage Expectations

You need to keep in mind the position of the manager. She wants success, obviously, because it's going to make her look good to her bosses. It might even be part of her performance goals and may affect her position or compensation in some serious ways. And there are the other teams in the company with which your manager might need to interact: salespeople who need to set expectations with customers (and for whom promises of the new project might be directly tied to sales numbers and commission-based compensation), support staff who need to be trained on the new project, Ops teams who need to allocate hardware and setup monitoring/alerting tools, various teams who need to adjust contracts, billing, metering, monitoring or other processes to keep track of the new project. Your manager, if she's any good, will be working with all these people and more to adequately set expectations and to keep all of this distracting chatter away from the ears of the developers. Your manager has to set expectations for all these people, and you need to help set hers. 

A little bit of empathy for other people goes a long way. Understanding that your boss has to communicate with external forces just as much as she needs to communicate with you is part of it. If you make a promise to your boss, and she has forward those promises on to the other teams and stakeholders, when something falls through it's your boss who's stuck eating a bowl of shit. Work with your boss on messaging. Try to paint a clear and realistic portrait of your status and your projections, and make sure that your boss has the information necessary to accurately share that message with others. Again, a bad boss will just say whatever she wants and ignore your information. That's her prerogative as a manager. Again, all you can do is try to provide the best, most accurate information to help manage expectations.

It's also worth understanding that a surprise failure is often worse than a planned failure. If you give people the expectation of failure, they can start to hedge bets and make compensations, so the eventual failure won't sting quite as much. **You should always under-promise and over-deliver** instead of the other way around. 

## On Bad Managers

Some managers are just lousy at what they do. We've all known people like this. A good manager will work with the team to try and find a path to success. A bad manager will dictate success, and blame everybody else when success doesn't just appear out of thin air. A good manager will have contingencies and backup plans ready in case things go wrong, and will always make sure to have a little bit of slack in the schedule to absorb unforeseen events. A bad manager won't have a backup plan, and isn't prepared for anything outside the happy path. Bad managers treat initial estimates like rock-solid promises, and then get angry when they turn out not to be accurate. A good manager will communicate with other teams and protect her developers from too much distraction. A bad manager will throw her team members under the bus to deflect from her own failings.

You may just have a bad manager. In that case you need to just tolerate as much as you can until it's time to find a new job. You may have a better manager than you realize, in which case you should try to work with her instead of constantly fighting against her. Give honest information, work to find compromise when things aren't progressing as expected, and help to manage expectations so there are no unpleasant surprises.

You might get fired, sure. People don't always like to hear bad information and won't always respond the way you think they should. If you do end up walking out the door just remember that the company is losing an employee who tried to be honest and manage a bad situation, while you have lost a crappy boss. It seems like a winning scenario to me.