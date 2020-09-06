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

> Note: `weaver-test` is built on top of `cats-effect` and `fs2`. If you're using `zio`, it has a module for `zio` integration; that being said, you should also consider looking at [`zio-test`](https://zio.dev/docs/usecases/usecases_testing).

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
## Stream

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

I am not fully convinced this is great. At the very least it's a pedagogical problem, since an explanation was needed. And it does smell a bit dynamically typed.

An alternative would be provide two separate functions, say `test` and `testM`, for declaring non-effectful and effectful tests. This is the approach `zio-test` takes, and one that `weaver-test-extra` might adopt in an upcoming version.

# Composing test values

As we pointed out, since tests are just values, we can compose and manipulate them in the usual ways.

Let's write a library function taking a list of expectations, and returning an expectation which passes if all the given expecations pass:

```scala
import cats.implicits._
import cats.data.NonEmptyList
import cats.effect.IO
import weaver.Expectations

package object example {
   def expectAll(xs: NonEmptyList[IO[Expectations]]): IO[Expectations] =  {
    xs.sequence.map(_.fold)
  }
}
```

This was easy since `Expectations` forms a monoid (by default, under `AND` / multiplicative semantics). 

We can write the dual of that by selecting the monoid with additive semantics:

```scala
def expectSome(xs: NonEmptyList[IO[Expectations]]): IO[Expectations] =  {
    xs.sequence.map { xs =>
      xs.map(Additive(_)).fold
    }.map(Additive.unwrap)
  }
```

Let's put that to use in our contrived example suite:

```scala
package io.github.dimitarg.example

import cats.effect.IO
import fs2.Stream
import weaver.pure._
import cats.data.NonEmptyList

object Examples extends Suite {

  override def suitesStream: Stream[IO,RTest[Unit]] = Stream(
      test("all expectations be true") {
          expectAll(
              NonEmptyList.of(
                expect(1 == 1),
                expect(2 == 2),
                expect(3 == 3),
              )
          )
      },
      test("at least one expectation must be true") {
          expectSome(
              NonEmptyList.of(
                expect(1 == 5),
                expect(2 == 6),
                expect(3 == 3),
              )
          )
      }
  )
  
}
```

```
io.github.dimitarg.example.Examples
+ all expectations be true
+ at least one expectation must be true


Execution took 17ms
2 tests, 2 passed
All tests in io.github.dimitarg.example.Examples passed
```


Let's try failing one of the tests:

```scala
test("at least one expectation must be true") {
          expectSome(
              NonEmptyList.of(
                expect(1 == 5),
                expect(2 == 6),
                expect(3 == 100),
              )
          )
      }
```

```
- at least one expectation must be true

 [1] assertion failed (src/test/scala/io/github/dimitarg/example/Examples.scala:23)
 [1] 
 [1] expect(1 == 5),

 [2] assertion failed (src/test/scala/io/github/dimitarg/example/Examples.scala:24)
 [2] 
 [2] expect(2 == 6),

 [3] assertion failed (src/test/scala/io/github/dimitarg/example/Examples.scala:25)
 [3] 
 [3] expect(3 == 100),
```

Neato.

See what's happening here? We set out to write a function operating on test values. We wrote the obvious code, did so using the obvious and familiar tools, and the code does what we expect. We used zero percent "test DSL" and "framework functions"  in the process. 

In other words, **we approached this programming task in the way we would aproach any other programming task**, and that worked! This is the world we want to live in.

# Manipulating test values

Let's set out and write some frameworky features for our testing library.

## Timeout

Can we easily implement a timeout function for tests?

We don't have to! We're already in `IO`, no need to reinvent the wheel.

```scala
test("timeout") {
  expect(1==1)
    .timeout(10.seconds)
}
```

## Flake

In general I think this is a poor idea, but for demonstration purposes, let's write a function which repeats a flaky test up to a certain nuber of times if it does not succeed.

(*It's a poor idea because if your test is flaky, the way to go is to investigate and fix it, and not write code to work around it. It's a slippery slope to cut corners in test infrastructure.*)

```scala
def flaky(attempts: Int)(x: IO[Expectations]): IO[Expectations] = {
      if(attempts<1) {
          x
      } else {
          x.attempt.flatMap(
            _.fold[IO[Expectations]](
              _ => flaky(attempts-1)(x),
                result => {
                  if(result.run.isValid) {
                    result.pure[IO]
                  } else {
                    flaky(attempts-1)(x)  
                  }  
                }  
              )
          )
      }
  }
```

```
test("flaky") {
  flaky(attempts = 10000) {
    IO(System.currentTimeMillis()).map { now =>
      expect(now % 2 == 0)  
    }
  }
}
```

## Ignore a test

Easy. (This already exists in `weaver-test`, but just for example's sake)

```scala
def ignored[A](reason: String)(x: RTest[A])(implicit loc: SourceLocation): RTest[A] = RTest(
      x.name, _ => IO.raiseError(new IgnoredException(reason.some, loc))
  )
```

```scala
ignored("too lazy to fix")(test("this will fail"){
  expect(1 == 3)
})
```

```
- this will fail !!! IGNORED !!!
  too lazy to fix (src/test/scala/io/github/dimitarg/example/Examples.scala:41)
```

The list goes on. The point being, since we work in `IO`, and `Expectations` is just data, we can manipulate individual tests any way we like.

# Composing suite values

Since suites are `fs2.Stream` values, we can compose and manipulate suites in the usual ways we work with 
`fs2.Stream`s.

For example, let's try to implement a "table-driven" test. I.e. run a multitude of tests generated by a table
of the following form

```
SCENARIO NAME  | INPUT | EXPECTED RESULT
========================================
Foo            | 42    | "Shrubbery"
Bar            | 86    | "KTHX"
...
```
, that was helpfully provided by your product owner.

The code under test rounds an electricity pay-as-you-go customer's balance in, up to penny, in their favour.
- If their balance is positive, i.e. they have more money on their account than they've used, we round up, giving them more money
- If their balance is negative, i.e. they are in debt, we round down,
giving them less debt

```scala
import weaver.pure._
import fs2._
import cats.effect.IO
import scala.math.BigDecimal.RoundingMode
import scala.concurrent.duration._

object RoundingSpec extends Suite {

  final case class EnergyBalance(value: BigDecimal)
  final case class Pence(value: Int)

  def roundInFavourOfCustomer(balance: EnergyBalance): Pence = {
      val roundingMode = if (balance.value >= 0) {
          RoundingMode.UP
      } else {
          RoundingMode.DOWN
      }

      val rounded = balance.value.setScale(2, roundingMode)
      val pence = (rounded * 100).toIntExact
      Pence(pence)
  }

  final case class TestScenario(scenarioName: String, energyBalance: EnergyBalance, expectedResult: Pence)

  val testData: Stream[Pure, TestScenario] = Stream(
    TestScenario("positive - nothing to round", EnergyBalance(2.49)   ,  Pence(249)),
    TestScenario("positive rounds up",          EnergyBalance(2.494)  ,  Pence(250)),
    TestScenario("positive rounds up - 2",      EnergyBalance(2.491)  ,  Pence(250)),
    TestScenario("negative - nothing to round", EnergyBalance(-2.49)  ,  Pence(-249)),
    TestScenario("negative rounds down",        EnergyBalance(-2.491) ,  Pence(-249)),
    TestScenario("negative rounds down - 2",    EnergyBalance(-2.499) ,  Pence(-249))
  )

  override def suitesStream: Stream[IO,RTest[Unit]] = testData
    .covary[IO]
    .map(x => test(x.scenarioName)(
        expect(roundInFavourOfCustomer(x.energyBalance) == x.expectedResult)
    ))
    .timeout(5.seconds)

}
```

We start by noting that a specification table is a `Stream` of values, in this case `testData: Stream[Pure, TestScenario]`. Then we map each element of the table to a test, which expects that rounding the row's
energy balance results in the expected amount of pence.

```
io.github.dimitarg.example.RoundingSpec
+ positive - nothing to round
+ positive rounds up
+ positive rounds up - 2
+ negative - nothing to round
+ negative rounds down
+ negative rounds down - 2

Execution took 21ms
6 tests, 6 passed
All tests in io.github.dimitarg.example.RoundingSpec passed
```

Nice. Table driven tests in 1 line of code - `fs2.Stream.map`. Consider `scalatest`, where we would have needed
framework support for this.


# "Test fixtures"

# Recap