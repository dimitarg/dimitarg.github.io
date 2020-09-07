---
title: "Purely functional testing in Scala"
date: 2020-09-06T00:00:00Z
published: true
categories:
  - Functional Programming
  - Testing
tags:
  - fp
---

Many projects written in Scala have now adopted the principles of purely functional programming. These projects are built in the safe subset of the language (also known as `scalazzi`). They utilise libraries such as `cats-effect`, `scalaz` or `zio` in order to be able to talk about computational effects without sacrificing referential transparency, parametricity and local reasoning.

However, for almost all these projects, the above claim is only true for the "production" part of the codebase. Test code is riddled with side effects. The reason is the widely used testing frameworks insist that we program with side effects:

- Failing a test is fundamentally done by throwing an exception;
- APIs such as `protected def beforeAll : Unit` and friends insist that any sharing of values / resources between tests must be done via side effects and shared mutable state;
- Put more generally, tests are not **values**.


# Why is this a big deal?

1. By sacrificing referential transparency in tests, we lose the productivity gains stemming from functional programming. We can no longer reason
equationally, or apply the substitution principle when refactoring. We lose the tools of composition, which means tests become harder to write, less concise and less obviously correct than they should be.

2. We use a different programming model, and a different mental model when writing production code and when writing tests. This creates mental context
shifting and makes us waste time and energy - for no good reason other than "`scalatest`/ `x` insists we do that".

3. Since we have already committed that our production code is pure, we are now forced
to test pure code in an impure testing language. This creates an impedance mismatch which at the very least manifests as `unsafeRunX` all over the place. The mismatch becomes especially apparent, boilerplate prone and error prone in use cases such as resource management, i.e. test / suite setup and teardown and the such.

4. By admitting that production code must be pure, but it's okay for tests to rely on side effects, we give tests a status of a second-class citizen.

The last point is worth reiterating. It does not make sense to lower our standards when it comes to the code that establishes the correctness of our programs; yet we still allow ourselves to do that. 

And I think there's no good reasons we do, aside from inertia and the status quo of popular tools.

# Enter pure testing

There exists a testing library called [weaver-test](https://disneystreaming.github.io/weaver-test/) which allows us to test in a referentially transparent manner. 

We will be using that, but what's written here should apply to other libraries with similar design. The central ideas behind `weaver-test` are:

- Introduce a data type to describe assertions
- The result of an assertion is then a value of this data type
- A `Test` is a value which computes assertions, potentially in `IO`
- A test suite is a collection of tests. It has type `fs2.Stream[IO, Test]`


## `weaver-test` basics

> Note: `weaver-test` is built on top of `cats-effect` and `fs2`. If you're using `zio`, it has a module for `zio` integration; that being said, you should also consider looking at [`zio-test`](https://zio.dev/docs/usecases/usecases_testing).

A test is a function which returns a value of type `Expectations` indicating whether the test succeeded or not:

```scala
case class Expectations(val run: ValidatedNel[AssertionException, Unit])
```

`Expectations` forms two monoids via `and` and `or` semantics, and can additionally be manipulated via the structure of `Validated` / `ValidatedNel`.

In addition, tests in general are allowed to peform `IO`. A test then has type

```scala
someTest: IO[Expectations]
```

> Side note: yet more generally, a test has type `F[Expectations]` for some type `F` from the `cats-effect` type hierarchy. In practice, that `F` is constrained to `ConcurrentEffect`. Since `ConcurrentEffect` is "morally `IO`", we will skip ceremonies, and postulate that a test has type `IO[Expectations]`.

This means that even if the code under test, or the test setup code is in `IO` (or some transformer stack containing `IO`), we don't have to resort to `unsafeRunX` in order to write the test, and that we can compose test values via `IO`, as well as via the applicative / monadic / monoidal structure of `Expectations` itself.

That is to say, we can now write test code the same way we write any other code - via the tools of functional programming!

## Going the last mile

Tests in `weaver-test` are values. However, when using the *default* API exposed, making sure a test is executed is still side effectful.

In the following snippet:

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

We'd like this to return a value instead.

Luckily, this problem is not inherent to the programming model of the library. I wrote a micro-lib [`weaver-test-extra`](https://github.com/dimitarg/weaver-test-extra) to address it. We will be using that in addition to `weaver-test` in all the below examples. It contains [nearly no code](https://github.com/dimitarg/weaver-test-extra/tree/60523cbe2fd58347ce3ab4fa5566a7f273dc9dd2/src/main/scala/weaver/pure) - you could write it yourself if you wanted, and probably do better. 

The point is, we now don't have to worry how side-effectful registrations might interact with regular code and break compositionality. It's now regular code all the way down.

Let's write some code to get a feel for purely functional testing, and what it buys us.

(All the following code is [available on github](https://github.com/dimitarg/weaver-test-examples).)

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
def test(name: String)(run: IO[Expectations]): RTest[Unit] = ...
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

## Digression - type trickery

`expect(1 == 1)` has type of `Expectations`, but we saw that `test` expects `IO[Expectations]` as input. Why does our code typecheck?

By importing `weaver.pure._`, we brought into scope

```scala
implicit def expectationsConversion(e: Expectations): IO[Expectations] =
    e.pure[IO]
```

The same code exists in vanilla `weaver-test`. This is done so that we can write effecful and non-effectful tests in the same manner. 

*I am not fully convinced this is great. At the very least it's a pedagogical problem, since an explanation is needed. And it does smell a bit dynamically typed.*

An alternative would be provide two separate functions, say `test` and `testM`, for declaring non-effectful and effectful tests. This is the approach `zio-test` currently takes, and one that `weaver-test-extra` might adopt in an upcoming version.

# Composing test values

As we pointed out, since tests are just values, we can compose and manipulate them in the usual ways.

Let's write a function taking a list of expectations, and returns an expectation that passes if all the given expecations pass:

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

Let's put that to use in an example:

```scala
package io.github.dimitarg.example

import cats.effect.IO
import fs2.Stream
import weaver.pure._
import cats.data.NonEmptyList

object Examples extends Suite {

  override def suitesStream: Stream[IO,RTest[Unit]] = Stream(
      test("all expectations must be true") {
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
+ all expectations must be true
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

An interesting thing is happening here. We set out to write a function operating on test values. We wrote the obvious code, did so using the obvious and familiar tools, and the code does what we expect. We used zero percent "test DSL" and "framework functions"  in the process. 

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

```scala
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

Since suites are `fs2.Stream` values, we can compose and manipulate suites in the usual ways we work with `fs2.Stream`s.

For example, let's try to implement a "table-driven" test. I.e. run a multitude of tests generated by a table
of the following form

```
SCENARIO NAME  | INPUT | EXPECTED RESULT
========================================
Foo            | 42    | "Shrubbery"
Bar            | 86    | "KTHX"
...
```
, that was helpfully provided by our product owner.

The code under test rounds an account fund balance up to the penny, in the account's favour.
- If the account balance is positive, we round up,  potentially giving them more money
- If their balance is negative, i.e. they are in debt, we round down,
giving them less debt

Let's start with the code under test.

```scala
final case class Balance(value: BigDecimal)
final case class Pence(value: Int)

def roundInFavourOfAccount(balance: Balance): Pence = {
  val roundingMode = if (balance.value >= 0) {
    RoundingMode.UP
  } else {
    RoundingMode.DOWN
  }

  val rounded = balance.value.setScale(2, roundingMode)
  val pence = (rounded * 100).toIntExact
  Pence(pence)
}
```

Next, a data type to model a row in our specification table, containing scenario name, input and expected result.

```scala
final case class TestScenario(
  scenarioName: String,
  balance: Balance, expectedResult: Pence
)
```

Then, our specification table becomes:

```scala
val testData: Stream[Pure, TestScenario] = Stream(
  TestScenario("positive - nothing to round", Balance( 2.49)  ,  Pence( 249)),
  TestScenario("positive rounds up",          Balance( 2.494) ,  Pence( 250)),
  TestScenario("positive rounds up - 2",      Balance( 2.491) ,  Pence( 250)),
  TestScenario("negative - nothing to round", Balance(-2.49)  ,  Pence(-249)),
  TestScenario("negative rounds down",        Balance(-2.491) ,  Pence(-249)),
  TestScenario("negative rounds down - 2",    Balance(-2.499) ,  Pence(-249))
)
```

We can map each element of this stream to a test that calls the function under test with the row's input,
and expects the expected result, and we'd get back a test suite. Here is the full listing:


```scala
package io.github.dimitarg.example

import weaver.pure._
import fs2._
import cats.effect.IO
import scala.math.BigDecimal.RoundingMode
import scala.concurrent.duration._

object RoundingSpec extends Suite {

  final case class Balance(value: BigDecimal)
  final case class Pence(value: Int)

  def roundInFavourOfAccount(balance: Balance): Pence = {
    val roundingMode = if (balance.value >= 0) {
      RoundingMode.UP
    } else {
      RoundingMode.DOWN
    }

    val rounded = balance.value.setScale(2, roundingMode)
    val pence = (rounded * 100).toIntExact
    Pence(pence)
  }

  final case class TestScenario(scenarioName: String, balance: Balance, expectedResult: Pence)

  val testData: Stream[Pure, TestScenario] = Stream(
    TestScenario("positive - nothing to round", Balance(2.49)   ,  Pence(249)),
    TestScenario("positive rounds up",          Balance(2.494)  ,  Pence(250)),
    TestScenario("positive rounds up - 2",      Balance(2.491)  ,  Pence(250)),
    TestScenario("negative - nothing to round", Balance(-2.49)  ,  Pence(-249)),
    TestScenario("negative rounds down",        Balance(-2.491) ,  Pence(-249)),
    TestScenario("negative rounds down - 2",    Balance(-2.499) ,  Pence(-249))
  )

  override def suitesStream: Stream[IO,RTest[Unit]] = testData
    .covary[IO]
    .map(x => test(x.scenarioName)(
      expect(roundInFavourOfAccount(x.balance) == x.expectedResult)
    ))
    .timeout(5.seconds)
}
```

Does that work?

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

Nice. Table driven tests in 1 line of code - `fs2.Stream.map`. Consider `scalatest`, where we would have needed framework support for this.

# "Test fixtures"

Finally, we'll look at test fixtures, a.k.a. the dreaded `before`/`after` and `beforeAll` / `afterAll`.

A question comes up a lot during tests, especially integration tests: how do we allocate some sort of resource needed by a test (or the program under test), and safely dispose of it afterwards? We might want to allocate said resource before each test; or we might want to reuse it throughout a whole test suite; or we might want to have a combination of the two.

The answer of traditional test frameworks is "side effects, plus framework lifecycle hooks such as `beforeXXX` / `afterXXX`". This is unfortunate and is expecially problematic when the code under test is itself pure and manages resources via functional patterns (the impedance mismatch we talked about at the beginning of the article.)

Usually in such a situation, one ends up using the unsafe primitive `cats.effect.Resource.allocated`, in a combination with mutable state in which to store the acquire and release actions; and invoke those actions
side effectfully in framework hooks. If you've written such code, you know it's a mess.

The answer once testing becomes pure is "just use `cats.effect.Resource`" (or whatever the resource management type of your effect system is). Since we already have `IO`, `Resource` and `Stream` at our disposal, we don't
need "framework support" to address such use cases.

Let's give an example. We'll start off with coming up with some imaginary resource

```scala
final case class DatabaseConnection(value: String)
```

, and a function to conjure it

```scala
def mkConnection(value: String): Resource[IO, DatabaseConnection] = for {
  _ <- Resource.liftF(IO(println(s"acquiring connection: $value")))
  result <- Resource.pure(DatabaseConnection(value))
} yield result
```

Next, a helper function to declare a test which will expect that a connection
has an expected value.

```scala
def connTest(conn: DatabaseConnection)(expected: String): IO[Expectations] = for {
  _ <- IO(s"got connection: $conn")
} yield expect(conn.value == expected)
```

Let's create a couple of tests that will use a "shared database connection".

```scala
val sharedConnectionTests: Stream[IO, RTest[Unit]] = 
  Stream.resource(mkConnection("shared-conn")).flatMap { conn =>
    Stream(
      test("shared connection test")(connTest(conn)("shared-conn")),
      test("shared connection - another test")(connTest(conn)("shared-conn"))
    )
  }
```
(both tests expect the connection passed to be `"shared-conn"` and will otherwise fail).

That was easy, `Stream.resource` gives us a single-element stream of that resource. `flatMap` gives us access to the emitted resouce, and we construct a stream of tests that have access to it.

Now let's spin up a couple of tests that use their own, isolated connection.
```scala
val ownConnectionTests: Stream[IO, RTest[Unit]] = 
  Stream(
    test("own connection - some test") {
      mkConnection("foo-conn").use { conn =>
        connTest(conn)("foo-conn")
      }
    },
    test("own connection - another test") {
      mkConnection("bar-conn").use { conn =>
        connTest(conn)("bar-conn")
      }
    }
  )
```
Ok, that's just `Resource.use`.

Finally, let's compose our suite:

```scala
override def suitesStream: Stream[IO,RTest[Unit]] =
    sharedConnectionTests ++ ownConnectionTests
```

Does it work?

(full example [here](https://github.com/dimitarg/weaver-test-examples/blob/ecb506eae8dad4dd05701a39d7844c3c38f502ca/src/test/scala/io/github/dimitarg/example/ResourceExample.scala)).


```
io.github.dimitarg.example.ResourceExample
acquiring connection: shared-conn
acquiring connection: foo-conn
acquiring connection: bar-conn
+ shared connection test
+ shared connection - another test
+ own connection - some test
+ own connection - another test

Execution took 39ms
4 tests, 4 passed
All tests in io.github.dimitarg.example.ResourceExample passed
```

There. We now have test and suite resource management - with 0 lines of "framework code"! This is again a consequence of the fact that writing test programs is just writing programs - if your testing library doesn't get in the way.

## `ResourceSuite`

In integration tests, a pattern that comes up often is "allocate resources / dependencies, bootstrap system under test,
execute a bunch of tests against it, sharing those resources, clean up". Since it comes up often,
`weaver.pure` has explicit support for it. (though as you saw in the previous paragraph, you'll be perfrectly fine writing that on your own)

You can write such a test via extending `RSuite` instead of `Suite`.

```scala
package io.github.dimitarg.example

import weaver.pure._
import cats.implicits._
import fs2.Stream
import cats.effect.Resource
import cats.effect.IO

object ResourceSuiteExample extends RSuite {

  final case class DatabaseConnection(value: String)

  override type R = DatabaseConnection

  override def sharedResource: Resource[IO, DatabaseConnection] = for {
    _ <- Resource.liftF(IO(println(s"acquiring shared connection")))
    result <- Resource.pure[IO, DatabaseConnection](DatabaseConnection("shared-conn"))
  } yield result


  override def suitesStream: fs2.Stream[IO,RTest[DatabaseConnection]] = Stream(
    rTest("some test") { conn =>
      IO(println(s"got connection $conn")) >>
        expect(1 == 1)
    },
    rTest("some other test") { conn =>
      IO(println(s"got connection $conn")) >>
        expect(2 == 2)
    }
  )
}
```

Here, we needed to 

- specify the type of our resource, `override type R = DatabaseConnection`
- describe how to acquire it, `override def sharedResource: Resource[IO, DatabaseConnection] = ...`
- The type of our suite becomes `fs2.Stream[IO,RTest[DatabaseConnection]]` (up until now we were working with `fs2.Stream[IO,RTest[Unit]]`)

`RTest[DatabaseConnection]` says that the test has an input parameter of type `DatabaseConnection`. You create a test with input by using `rTest` instead of `test`.

>You can another example of using `RSuite` [here](https://github.com/dimitarg/weaver-test-extra#using-subsets-of-a-shared-resource-across-multiple-modules), dealing with multiple types of resources.

In any case, this is just one way to approach the problem. `fs2` gives you superpowers, and you are free to come up with your own approach, better suiting the needs of your project.

# Conclusion

Once we remove side effects from testing, we regain back compositionality. This means regaining back productivity, because when the testing library gets out of the way, writing test programs becomes writing regular programs.

Being more productive when writing tests will mean testing more thoroughly, writing less incorrect tests and flaky tests, and catching more bugs before they make it into production.

The vehicle for this exists, and I encourage you to give purely functional testing a try now!

Since libraries such as `weaver-test` and `zio-test` can be used alongside your legacy tests, you can start reaping the benefits of functional testing immediately.
