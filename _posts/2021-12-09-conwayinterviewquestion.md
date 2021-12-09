---
layout: post
categories: [JobHunt]
title: Conway's Game Interview Question
---

I saw a post on Reddit today from one of the Engineering Managers and [an interview question he liked asking candidates](https://alexgolec.dev/reddit-interview-problems-the-game-of-life/). The question was an implementation of [Conway's Game of Life]() and some optimizations around calculating state.

> Your task for the interview is: given an array of arrays representing the board state ( 'x' for alive and '' for "dead"), compute the next state.

Honestly I'm not too thrilled with this as an interview question, though I can see why some people might like it. First, the part I like:

> Let's dive into that implementation complexity. We can decompose this problem into two parts: computing a new alive/dead status for an individual cell, and then using that per-cell computation to compute the next state for the entire board.
> I say this in an offhand way, but being able to recognize and separate those two tasks is already an achievement.

I like this aspect of the question because it does show a person's ability to decompose a big problem into a series of smaller problems, and focus on one of these at at time. It's interesting to see if a person can decompose it, or if they think about the whole thing as a big complex mess of interlocking parts. 

There are two things I don't like about this question:

1. There is very little real-world relevancy here. There just aren't too many companies either implementing Conway's Game of Life, or solving problems which are closely related to it. That is, what you're learning about how the candidate solves this problem might not correlate very well to what you need them to do when they're on the job. 
2. Some people are going to have prior knowledge of Conway's Game of Life, and these people are going to have an intuitive advantage and higher level of comfort dealing with the problem than somebody who hasn't. Many candidates already have a problem with nerves when they're interviewing, doing something like this will just exacerbate this for some and ease it for others, without any indication of which is which.

To the second point I will admit it might be interesting to judge a person's performance in this question against their prior experience with Conway's Game of Life, but the problem is that you don't know their prior experience. So you've created an uneven playing floor and you don't know from which position any individual candidate is starting. A person who has significant prior experience with the Game but barely scrapes through the question is more of a disappointment than a person with no prior knowledge of it whatsoever but manages to cobble together the same solution. But, again, this question doesn't really admit that. At least not in an interviewing setting, which is where we are.

A good interview question should check the following boxes:

1. It should be at least tangentially relevant to the work you're expecting the person to perform assuming they get hired, and should hopefully bear some resemblance to work they've been doing at their previous position (assuming, of course, this isn't hiring an entry-level position).
2. It shouldn't require some kind of trick, surprise insight, or "Eureka" moment to get right. You can't demand, script or schedule deep flashes of insight, so your question shouldn't depend on them.
3. It should not be related to specific prior knowledge, so every candidate can begin the question from an equal footing. Also it shouldn't make a candidate more nervous to have to solve a problem that they've heard about before but never studied. 

Slightly better than this question about Conway's Game of Life would be to ask a similar question about the Levenshtein Distance Algorithm (an oft-mentioned favorite of mine), which has a very similar implementation (2D grid of state), admits a very similar optimization (reusing old arrays instead of maintaining the entire result grid), is likely to be relevant to a lot of business interests, and is obscure enough that fewer candidates will have heard about it in school. Levenshtein still isn't perfect though because some candidates invariably *have heard about it before* which leads to unequal footing again. I'm not recommending this as an interview question, simply pointing out that this question is slightly better than the worse question. 

## The Turtle Problem

There was a question that a previous boss of mine used to ask candidates which I also never liked. That question goes like this:

> You have a turtle on an infinite grid. For each input integer value, the turtle walks forward that number of steps and then turns left. Given an input array of integer values representing the number of steps the turtle walks between turns, calculate whether the turtle ever crosses his own path and on which move he does.

So, given an input like `[3, 5, 2, 7]` the turtle walks forward 3 steps and turns left, walks 5 steps and turns left, walks 2 steps and turns left, then walks 7 steps. If you draw this path out on a piece of graph paper it should be obvious that the turtle crosses his own path on the fourth move.

The thing with this question, and the reason why I didn't like it, is that it requires a very specific insight to get right. Actually a complete solution requires a few specific pieces of insight to get completely right. 

If the turtle is always turning left there are basically two cases where a crossing might happen. First, if the turtle is "spiralling out" with line segments getting progressively longer, the turtle will only cross if the current line crosses the line from 5 moves ago. So, for example, the input sequence `[2, 1, 3, 2, 2, 2]` crosses when the sixth move crosses the first move. Second, if the turtle is "spiralling in" with line segments progressively getting shorter, the turtle only crosses his previous path if the current move intersects the line from 3 moves ago. `[4, 4, 4, 3, 2, 5]` is an example where the sixth move crosses the third move. Knowing that for each given move you only need to check for intersections with two previous moves is the key to optimizing this solution. A naive solution might check each new line against each previous line, leading to quadratic performance. The optimal solution will only check against specific lines, leading to linear performance. 

The expression for this solution gets very messy very quickly because of all the array indexing that happens. If the input is `M` and the current move index is `n` we calculate a solution with an expression something like this:

```csharp
if ((M[n] >= M[n-4] - M[n-6] && M[n-1] >= M[n-3] - M[n-5]) ||
    (M[n] >= M[n-2] && M[n-1] <= M[n-3]))
    return true;
```

(Keep in mind that I threw this solution together off the top of my head without testing so it might be wrong, but the "real" solution is at least this messy. I've seen so many implementations of it and few of them ever got better or more readable than that.)

There's also two more pieces of insight you must have for a "complete" solution. The first is that there's no possibility to cross it's own path before the 4th move, so you can simply start your loop at `n = 4`. The second is that there is a degenerate case on the fifth move where it's possible to come into the origin point from the back, overlapping the first line which isn't perpendicular so it needs to be checked separately. 

A completely optimal solution short-circuits with short inputs, then it unrolls the loop to execute the first 3 moves immediately, checks the fourth move normally, checks the fifth move for the special degenerate case, and then loops over all subsequent moves checking only move `N-3` and `N-5` for intersections.

The reasons why I don't like this problem are because it has absolutely nothing to do with the work we actually needed the candidate to do, it has absolutely nothing to do with the work they were doing at their previous employer, it requires a flash of insight to find the trick to get the solution to be "correct", there are too many edge cases that could lead to failures, and even when the solution was provided it was often such a ridiculous mess of code that it was impossible to gain insight into the person's style or code quality. 

## Nerves and Bozos

Anybody who has done any amount of programmer interviewing will be able to tell you that there are lots of bozos out there. There are people who will come into your interview and waste an hour of your time despite not knowing the first thing about programming. There are people for whom FizzBuzz is an intractable problem, for whom finding duplicate letters in a string is impossible, for whom adding every fifth integer into a sum is complete and total gibberish. These bozos exist in large numbers and you need to be aggressive about weeding them out. 

On the other hand there are plenty of people who might actually be good programmers and good workers, who have problems with nerves or anxieties which cause them to under-perform in interviews. That is, being put on the spot in a big room with a whiteboard and a few senior staff isn't always a comfortable place to be. It's not the best environment to elicit cogent responses from all people. Being too aggressive about eliminating the bozos will also create a string of false-negatives among these candidates, which is only going to increase the amount of time and energy you spend trying to find a winner. 

Finding ways to help calm the nerves and pull out the best in people is one of the trickiest parts of any interview. Tricky "gotcha" questions will only exacerbate this problem. Simple, straight-forward questions with real-world relevancy will help. Giving candidates the problems to take home and work on in their own time will also help. Then when they come on-site you can use the time to review the solution instead of watching over their shoulder while they struggle through it.

You still need programming questions. You still need to protect yourself and your company from the bozos. Just make sure the way you're doing it isn't also eliminating people who are good workers but bad interviewers.

## End Goal

The end goal of any interview question is to try and make a very specific decision: Will this candidate be a good match for the open position and for my team? If your question doesn't have obvious relevancy to this decision, or if you're creating a situation where otherwise-good candidates are put at a disadvantage while otherwise-bad candidates are made to look unusually appealing, you should dump that question on the floor. Interviewing programmers is like any other programming pursuit: you must continue to iterate and optimize over time, making sure your solution aligns as close as possible to the requirements. 