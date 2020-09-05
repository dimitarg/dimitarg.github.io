---
title: "Purely functional testing in Scala"
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

And I think there's no good reasons we do, besides inertia and the status quo of popular tools.

# Enter pure testing

There exists a testing library called [weaver-test](https://disneystreaming.github.io/weaver-test/) which allows us to test in a referentially transparent manner. 

> Note: `weaver-test` is built on top of `cats-effect` and `fs2`. If you're using `zio`, consider looking at [`zio-test`](https://zio.dev/docs/usecases/usecases_testing).

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

The point is, we now don't have to worry how side-effectful registrations might interact with regular code and break compositionality. It's now regular code all the way down. Let's write some.

All the below code is [available on github](https://github.com/dimitarg/weaver-test-examples).

# Build setup

The examples have the following in the `build.sbt`:

```scala
resolvers += Resolver.bintrayRepo("dimitarg", "maven")
libraryDependencies +=  "io.github.dimitarg"  %%  "weaver-test-extra" % "0.3.0" % "test"

testFrameworks += new TestFramework("weaver.framework.TestFramework")
```

This pulls `weaver-test` into your test classpath, as well as the micro-library described above.

# Minimal example

Let's start by writing a minimal `weaver-test` example and breaking it down.

```scala
package io.github.dimitarg.example

import weaver.pure._
import cats.effect.IO
import fs2.Stream
object MinimalTest extends Suite {
  override def suitesStream: fs2.Stream[IO,RTest[Unit]] = Stream(
    test("reality is still in place") {
      expect(1 == 1)
    }
  )
}
```

First thing of note is that in order to start writing tests, a single import `import weaver.pure._` is required. Here, this brings in scope `Suite`, `test`, `expect` and a couple of implicits we'll discuss below.

Next off, we see that a suite is defined to be a value of type `fs2.Stream[IO,RTest[A]]`.

The datatype `RTest` comes from `weaver-test-extra`. Let's examine it:

```scala
final case class RTest[R](name: String, run: R => IO[Expectations])
```

This says a test has a name, returns `Expectations` and can perform `IO` while doing so.

In addition it can access an input parameter of type `R`. This comes in handy if you need multiple tests or suites to have access to some sort of shared environment (such as a suite-wide `Resource`).

Here we need no such environment, i.e. in our case `R=Unit`. We use the function `test` which returns `RTest[Unit]` 

```scala
def test(name: String)(run: IO[Expectations]): RTest[Unit] = ???
```

Let's now get back to the type of a suite, `fs2.Stream[IO,RTest[A]]`. This says that a suite is a `fs2.Stream` of tests. 

This is great! It means that whatever we can do with an `fs2.Stream`, we can do with a suite. Filtering stuff out, interleaving with extra tests, running multiple suites in parallel, providing suite-wide timeouts, performing effects in-between tests ... `Stream` is the limit. 

This means we now have three compositional tools at our disposal when working with tests:

- The monoidal, applicative and monadic structure of `Expectations`
- `IO` at the test level
- `fs2.Stream` at the suite level

## Error reporting

Let's make our test fail to see how error reporting looks like.

```scala
test("reality is still in place") {
  val x = 42
  expect(x == 1)
}
```

```
- reality is still in place
  assertion failed (src/test/scala/io/github/dimitarg/example/MinimalTest.scala:10)

  expect(x == 1)
         | |
         | false
         42

```

Neato.

## Type trickery

`expect(1 == 1)` has type of `Expectations`, but we saw that `test` expects `IO[Expectations]` as input. Why does our code typecheck?

By importing `weaver.pure._`, we brought into scope

```scala
implicit def expectationsConversion(e: Expectations): IO[Expectations] =
    e.pure[IO]
```

The same code exists in vanilla `weaver-test`. This is so that you can write effecful and non-effectful tests the same way. 

I am not fully convinced this is great. At the very least it's a pedagogical problem, since an explanation was needed.

An alternative would be provide two separate functions, say `test` and `testM`, for declaring non-effectful and effectful tests. This is the approach `zio-test` takes, and one that `weaver-test-extra` might adopt in an upcoming version.