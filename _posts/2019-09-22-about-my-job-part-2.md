---
title: "About my job (Part 2/2)"
date: 2019-09-22T13:30:00Z
published: false
categories:
  - Functional Programming
tags:
  - non-techie
  - fp
---

[Last time](2019-09-21-about-my-job.md), we established that software sucks, and that we must do better.
We must do better, because when computer programs fail, what is at stake is real people's time, money,
well-being, or worse, their lives.

(Let me be clear that by "we must do better", I also mean that **I, personally, must continuously attempt to do better.**)

How might we make computer programs better, i.e. less likely to fail?

# Programming got divorced

My dad is a construction engineer (and a good one, I might add), and so, I am keen to draw an analogy from construction again. Notice that buildings (mostly) do not fall apart; on the other 
hand almost all computer programs fail at one point or another.

So what is it that construction engineers do to make sure that buildings do not fall apart?
Well, lots of things, but the gist of it all is that when designing buildings, they observe the laws
of mathematics. More precisely, construction engineers observe the laws of physics, but [physics is
just applied mathematics](https://xkcd.com/435/) if you look under the hood.

> But don't computer programmers do the same? After all, you folks take mathematics exams to go into university, and study more maths in there?

Oh boy, would you be surprised! Sadly, the answer is - usually, no! **Programmers rarely follow the rules of mathematics when they design programs.** I think this is the primary reason most programs fail.

What we mostly do instead is we piece the program together, in the hand-wavy manner in which a painter will create a painting - putting a base, then some layers and figures together, and then some finishing touches - all based on "what feels right". We will then (hopefully) test the program a little bit, and if we don't discover glaring defects, we will declare that it looks good as far as we can tell, and be done with it.

So programmers work more like craftsmen than engineers, but then the programs will fail - because we forgot to strictly observe the laws of reality when crafting - which is OK for a Dali piece, but really,a poor way to write a program.

**But why?**

Why did programming get divorced from mathematics? And what do we do to bring them back together?
To talk about this, we should first make sure we understand what a program really **is**.
As promised last time, we will do it without technical jargon. Instead, we will draw some pictures.


