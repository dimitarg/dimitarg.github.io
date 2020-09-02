---
title: "Testing a purely functional codebase with pure functions"
date: 2020-09-02T00:00:00Z
published: false
categories:
  - Functional Programming
  - Testing
tags:
  - fp
---


Many projects written in Scala have now adopted the principles of purely functional programming. Those projects are written in the safe,
purely functional subset of the Scala programming language. They utilise libraries such as `cats-effect`, `scalaz` or `zio`, in order
to be able to talk about computational effects (such as interacting with the outside world, modifying global state, etc) without
sacrificing referential transparency, parametricity and local reasoning.


However, for almost all these projects, the above claim is only true for the "production" part of the codebase. Test code is riddled with side effects. The reason is the widely used testing frameworks insist that we program with side effects:

- Failing a test is fundamentally done by throwing an exception;
- APIs such as `protected def beforeAll : Unit` and friends insist that any sharing of values / resources between tests must be
done via side effects and shared mutable state;
- Put more generally, tests are not **values**.


In this article, I will
- Make the claim that this is not just a niusance, but a real problem;
- Show that can do better - that people have already done better, and there exists no good reasons to continue writing tests that are not values;
- Give plenty of code examples using a specific library

# Why is this a big deal?

1. By sacrificing referential transparency in tests, we lose the productivity gains coming from functional programming. In general, we can no longer reason
equationally, or apply the substitution principle when refactoring. Worst of all, we lose the tools of composition, which means tests become harder to write, less concise and less obviously correct than they should be.

2. We use a different programming model, and a different mental model when writing production code and when writing tests. This creates mental context
shifting, wasting time and energy - for no good reason other than "scalatest / x insists we do that".

3. Since we have already committed that our production code is pure, we are now forced
to test pure code in an impure testing language. This creates an impedance mismatch which at the very least manifests as `unsafeRunX` all over the place.
The mismatch becomes especially apparent, ugly and boilerplate-prone once you try do something more complicated, such as share resources across multiple tests or test suites.

4. By admitting that production code must be pure, but it's okay for tests to rely on side effects, we give tests a status of a second-class citizen.

Think about the last point for a while.

It's silly! How come we are allowed to lower our standards when it comes to the code that establishes the correctness of our programs?

I'm convinced it makes no sense to do so, and there exist 0 reasons to continue doing so, apart from inertia.


