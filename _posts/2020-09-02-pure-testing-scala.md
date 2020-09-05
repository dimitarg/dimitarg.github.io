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


# Why is this a big deal?

1. By sacrificing referential transparency in tests, we lose the productivity gains coming from functional programming. In general, we can no longer reason
equationally, or apply the substitution principle when refactoring. Worst of all, we lose the tools of composition, which means tests become harder to write, less concise and less obviously correct than they should be.

2. We use a different programming model, and a different mental model when writing production code and when writing tests. This creates mental context
shifting, wasting time and energy - for no good reason other than "scalatest / x insists we do that".

3. Since we have already committed that our production code is pure, we are now forced
to test pure code in an impure testing language. This creates an impedance mismatch which at the very least manifests as `unsafeRunX` all over the place. The mismatch becomes especially apparent, ugly and boilerplate-prone once you try do something more complicated, such as share resources across multiple tests or test suites; deal with a `cats.effect.Resource` in a `beforeAll` / `afterAll` setup, bootstrap and shutdown a whole `IOApp` safely and without leaking resources within a given testing scope, etc.

4. By admitting that production code must be pure, but it's okay for tests to rely on side effects, we give tests a status of a second-class citizen.

The last point is worth reiterating - it doesn't make sense to lower our standards when it comes to the code that establishes the correctness of our programs; but we still allow ourselves to do that. 

And I think there's no good reasons we do that, besides inertia and the status quo of popular tools.

# Enter pure testing

There exists a testing library called [weaver-test](https://disneystreaming.github.io/weaver-test/) which allows us to test in a referentially transparent manner. 

That is, a test it a function which returns a value of type `Expectations` indicating whether the test succeeded or not:

```scala
case class Expectations(val run: ValidatedNel[AssertionException, Unit])
```

`Expectations` forms two monoids via `and` and `or` semantics, and can additionally be manipulated via the structure of `Validated` / `ValidatedNel`.

In addition, tests in general are allowed to peform `IO`, and so a test in general has type 

```scala
someTest: IO[Expectations]
```

> Side note: yet more generally, a test has type `F[Expectations]` for some type `F` from the `cats-effect` type hierarchy. In practice, that `F` is constrained to `ConcurrentEffect`. Since `ConcurrentEffect` is "morally `IO`", we will skip ceremonies, and postulate that a test has type `IO[Expectations]`.

This means that even if the code under test, or the test setup code is in `IO` (or some transformer stack containing `IO`), we don't have to resort to `unsafeRunX` in order to write the test, and that we can compose test values via `IO`, as well as via the applicative / monadic / monoidal structure of `Expectations` itself.

That is to say, we can now write test code the same way we write any other code - via the tools of functional programming! 

If you're a haskell programmer, your reaction at this point probably is "well, yeah." But for Scala, I think this is a game changer, and I hope you're as excited about this as I am!


## Short digression - going the last mile

While tests in `weaver-test` are values, when using the default API, *making sure* a test is executed is side effectful. What we mean by this is that in the following snippet:

```scala
test("some test") {
  doSomething >>
    expect(42 == 42)
}
```

, the function `test` has type

```scala
def test(name: String)(run: IO[Expectations]): Unit
```
, where `Unit` indicates that a side effect is performed in order to register the passed test value with the framework.

This however is just a trait of the default `Suite` API and not inherent to the programming model. So I wrote a micro-library [`weaver-test-extra`](https://github.com/dimitarg/weaver-test-extra) overcoming that problem. We will be using this library in addition to `weaver-test` in all the below examples. The library contains [nearly no code](https://github.com/dimitarg/weaver-test-extra/tree/60523cbe2fd58347ce3ab4fa5566a7f273dc9dd2/src/main/scala/weaver/pure) - you could write it yourself if you wanted.