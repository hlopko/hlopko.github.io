---
layout: post
comments: true
title: "Knuth: Premature optimization is the root of all evil - the context."
description: ""
category:
tags: []
date: 2019-08-03
---

I came across the Knuth's paper from which the famous quote "Premature
optimization is the root of all evil" comes from. With more context, the message
is a bit more nuanced - see for yourself:

>
> The improvement in speed from Example 2 to Example 2a is only about 12%, and
> many people would pronounce that insignificant.  The conventional wisdom
> shared by many of today’s software engineers calls for ignoring efficiency in
> the small; but I believe this is simply an overreaction to the abuses they see
> being practiced by penny-wise-and-pound-foolish programmers, who can’t debug
> or maintain their “optimized” programs. In established engineering disciplines
> a 12% improvement, easily obtained, is never considered marginal; and I
> believe the same viewpoint should prevail in software engineering. Of course I
> wouldn’t bother making such optimizations on a one-shot job, but when it’s a
> question of preparing quality programs, I don’t want to restrict myself to
> tools that deny me such efficiencies.
>
> There is no doubt that the grail of efficiency leads to abuse. Programmers
> waste enormous amounts of time thinking about, or worrying about, the speed of
> noncritical parts of their programs, and these attempts at efficiency actually
> have a strong negative impact when debugging and maintenance are considered.
> We should forget about small efficiencies, say about 97% of the time:
> premature optimization is the root of all evil.
>
> Yet we should no pass up our opportunities in that critical 3%. A good
> programmer will not be lulled into complacency by such reasoning, he will be
> wise to look carefully at the critical code; but only after that code has been
> identified. It is often a mistake to make a priori judgments about what parts
> of a program are really critical, since the universal experience of
> programmers who have been using measurement tools has been that their
> intuitive guesses fail. After working with such tools for seven years, I’ve
> become convinced that all compilers written from now on should be designed to
> provide all programmers with feedback indicating what parts of their programs
> are costing the most; indeed, this feedback should be supplied automatically
> unless it has been specifically turned off.
>

So not premature, but uninformed optimization is the problem. Keep on
optimizing, but only where it matters :)

[Knuth, Donald E. "Structured Programming with go to Statements." ACM Computing
Surveys (CSUR) 6.4 (1974):
261-301.](http://cowboyprogramming.com/files/p261-knuth.pdf)
