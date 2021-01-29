---
layout: post
categories: [Design]
title: Reasoning About Least Surprise
---

I saw a great, concise blog post the other day titled "[Reasoning about Code is a Scam](https://www.sicpers.info/2021/01/reasoning-about-code-is-a-scam/)". Do yourself a favor and check it out if you haven't seen it already.

> More precisely, it’s a thought-terminating cliche. 
> ...
> It’s a scam.
> 
> Let’s start with the fact that people don’t think—sorry, reason—about things the same way. 

This is a really great point, and one that I don't think gets enough coverage and understanding. Different people think about code in different ways. Different people have different ideas about what "good" code looks like. I've known people who think about code in a very imperative way, and seem able to effectively reason about nested control structures more easily than an object graph. I've known people who prefer OO code over FP, or vice-versa, and can make strong arguments in support of their preference. I've known people who prefer short methods, long methods, methods with only a single `return` statement. Making code "better" for one person is very likely to make it worse for somebody else. Making a statement like "code is better" or "easier to reason about" or "...considered harmful" almost always feels like a painful over-generalization which often says more about the narrow-mindedness of the author than it says about programming as a whole. There are very few good rules which seem to really hold water and make code better whenever they're applied, the rest really boils down to personal style.

I myself have a very strong, refined sense of style. I know exactly what I think code should look like. I know how to make code "better" for my own consumption. There are plenty of people who disagree with me, however. I like small methods, but [not everybody agrees with me](https://copyconstruct.medium.com/small-functions-considered-harmful-91035d316c29) (at least, not always). I tend to like small, focused classes. Some people disagree with that. [Some people don't like classes at all](https://psc.informatik.uni-jena.de/publ/1992-Rechteck-Quadrat-jfhw.pdf). I like the Single Responsibility Principle, but [good luck getting people to agree on what that even means](http://whiteknight.github.io/2020/08/24/reconsidersrp.html).

Buried in this blog post is quite a gem of a quote:

> Code is a particular representation of, at best, yesterday’s understanding of the problem you’re trying to solve.

This is a perfect explanation of the need to constantly refactor code. Code is never really "done". It is never "perfect". The best you can do is get to a point where the difference in your understanding of the problem since yesterday isn't sufficiently large enough to justify editing the code again. That's assuming, of course, that you are continuing to think about the problem space and are refining your understanding of it on a regular basis. You are doing that, aren't you?

Honestly, I think the bigger goal isn't to improve "ability to reason about the code" but to satisfy the **[Principle of Least Surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)**:

> a component of a system should behave in a way that most users will expect it to behave.

Structure your code in a way that is consistent. Avoid [lava flows](http://antipatterns.com/lavaflow.htm) of half-implemented ideas and incomplete refactors. Even if a piece of code doesn't match your own personal style, so long as it is consistent with the rest of the system of which it is a part, other developers will be able to work on it with low levels of surprise or astonishment.

Code only becomes easy to reason about over time, with experience. A person who is working in a code base for a long time should be able to reason about that code effectively, and they can only do that if the code is consistent and unsurprising. 