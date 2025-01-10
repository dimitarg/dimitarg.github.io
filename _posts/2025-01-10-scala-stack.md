---
title: "A 2025 Scala stack for the functionally inclined"
date: 2023-01-09T10:00:00Z
published: true
categories:
  - Functional Programming
  - Scala
tags:
  - fp
  - cats-effect
  - fs2
---

I recently started a new Scala project, one which is greenfield. Building something new from the ground up poses the need - and opportunity - to make technology choices.

Scala is a language where there's usually many ways to achieving a goal, and the same can be said for Scala libraries. This is a good thing because it gives options, and somewhat daunting, because it's time-consuming to arrive at a set where all the pieces are good, and work well when put together.

Below we present an opinion piece of what a Scala stack can look like in 2025, if your favourite paradigm happens to be purely functional programming. Almost all of the options presented are not novel, and I've used them in services which are in production.

This is also an Ode to all the people who quietly spent countless hours to bring us - for free - all the amazing open-source libraries we use (and take for granted!) every day. Thank you!

# Language version

Prior to this exercise I hand't done any Scala 3, personally or professionally. I've been reading and writing code for a while now, so I figured picking up the new language version itself wouldn't be too much of a hassle. I was more worried whether all my libraries of choice would work with Scala 3, and in 2025 they all do, with a couple of exceptions I'll note in subsequent paragraphs. Those usually have adequate replacements, so for new **services**, I'd recommend defaulting to Scala 3.

The reason I recommend Scala 3 for greenfield, service (not library) development is

- For application / service programming, it appears to be at least a marginally better language than Scala 2. It's definitely not a worse one.
- It's going to take a somewhat experienced Scala 2 developer 5-20 hours to get started with Scala 3, which is negligible in the long run
- You'd probably be introducing future technical debt in your new project if you choose Scala 2 now.

This recommendation comes with the huge caveat that I don't have Scala 3 experience with an extremely large codebase. I've previously worked on a project that took an hour to compile, and I cannot say if that hour is going to become 50 minutes or 3 hours if they made the switch.

## Language changes 

I didn't find picking up language changes hard. My current personal and work projects try to stick to a relatively small subset of Scala that's purely functional, and all compile with `-Xsource:3`, so the transition was mostly painless. Here's what I gradually did while building my new project.

- Pick up new control flow syntax - not mandatory (with Scala `3.3.4`) and can be automated, as described later. This mostly boiled down to writing `if a then foo else bar` in place of `if (a) foo else bar`
- Stop writing optional braces - not mandatory and can be automated, as described later.
- Use `given / using` in place of `implicit`
- Use `extension` in place of `implicit class`
- Use `enum` in place of `sealed trait` whenever describing sum types. If my understanding is correct, this mostly works, however they currently don't have 100% feature parity. Specifically enums cannot nest enums. You can revert to `sealed trait` whenever facing issues, granted that's not great for consistency.

> The above is, of course, by no means an exhausting list of language changes and new language features in Scala 3. Specifically it leaves out any new and changed type-level and metaprogramming facilities. We're mostly focused on slightly boring application programming in this article.

## Cross-building own libraries

You might be in the position of maintaining libraries that need to be used across Scala 2 and Scala 3 services.

One way to solve this, if feasible for the code in question, is to code your libraries to a subset of Scala 2 that's also valid Scala 3. For me, this boiled down to `"-Xsource:3-cross"`, or, more completely:

```
libraryDependencies ++= {
  CrossVersion.partialVersion(scalaVersion.value) match {
    case Some((2, n)) =>
      List(
        compilerPlugin("org.typelevel" % "kind-projector" % "0.13.3" cross CrossVersion.full)
      )
    case _ =>
      Nil
  }
}

ThisBuild / scalacOptions ++= {
  CrossVersion.partialVersion(scalaVersion.value) match {
    case Some((2, 12 | 13)) => Seq("-Xsource:3-cross", "-P:kind-projector:underscore-placeholders")
    case _ => Nil
  }
}
```

If such a simplistic and limited approach does not work for you, take inspiration from established open source projects.

# Enforcing code style

## `sbt-tpolecat`

This turns on a number of useful compiler warnings, and by default, turns warnings to errors.

```
addSbtPlugin("org.typelevel" % "sbt-tpolecat" % "0.5.2")
```

## scalafmt

I find myself being inconsistent with regards to optional brace usage and new control flow syntax usage. The bigger the squad, the bigger the inconsistencies will be. You can enforce this via tooling. In my `.scalafmt.conf`, I've settled for

```
version = 3.8.3
maxColumn = 120
runner.dialect = scala3

rewrite.scala3.convertToNewSyntax = true
rewrite.scala3.removeOptionalBraces = true
```

## scalafix

I find myself faffing with imports a lot. Let tooling sort that instead. For example,

```
rules = [
  OrganizeImports
  RedundantSyntax
]
OrganizeImports.targetDialect = Scala3
OrganizeImports.removeUnused = false
OrganizeImports.groupedImports = Merge
```

# Core abstractions and effect system

Core abstractions buy us the capability to not repeat ourselves, and effect systems help us increase program correctness, build resource-safe and cancellation-safe code, and write parallel and concurrent programs in a concise and safe manner.

For core data types and abstractions, I've went with `cats`, and its companion `cats-effect` as an effect system.

The other notable alternative is `zio`, which provides core abstractions and an effect system in one bundle. My recommendation is to use whatever you and your team are most productive with. If you are not yet invested in either, your evaluation must take into account whether all libraries you need exist / work well with your choice - lest you need something niche which exists in only one of the ecosystems. 

I imagine in 99% of cases you'd be fine with either.

# Stream programming

`cats-effect`-based projects have settled on `fs2`. It's mature, feature-rich, has an amazing community on Discord, and libraries for many technologies you may need exist that are built on top of it. That last part is extremely handy, because it makes your whole stack compose together well, when individual pieces are picked with care.

# Testing

If you use an effect system, you want a test framework which is integrated with it. The alternative would be using a side-effecting test framework such as `ScalaTest`, but that forces you to write bridge code between pure expression land (your code under test) and side effect land - think lots of `allocated` and `unsafeRunSync`. This becomes very unwieldy very fast, and is easy to get wrong. It's not a great option.

I've settled for [`weaver-test`](https://github.com/disneystreaming/weaver-test), since it's based on `fs2` and composes nicely with the rest of the stack. It works well and I've used it on a number of projects. 

In addition I use a micro-library I'm the author of, [`weaver-test-extra`](https://github.com/dimitarg/weaver-test-extra), to aid in purely functional test setup and teardown. (See the repo readme for reasoning, use and limitations).

`weaver-test` is in the process of being [transferred](https://github.com/typelevel/governance/issues/114) from `disneystreaming` to `typelevel`. I expect this to be a good thing, as it might receive more traction and more maintenance.

## Integration testing

I.e. spinning up docker services.
Use [testcontainers-scala](https://github.com/testcontainers/testcontainers-scala). Use `cats.effect.Resource.fromAutoCloseable` to view a container as a `Resource`, which will let you achieve resource-safe setup and teardown.

## Property testing and random data generation

Use `scalacheck`. `weaver-test` provides `scalacheck` integration out of the box. 

If instead of a property test, you need an ad-hoc piece of random data for a regular test,

```scala
extension [A](gen: Gen[A])
  def sampleF[F[_]: Sync]: F[A] = Sync[F].delay(gen.sample.get)
  def sampleIO: IO[A] = sampleF[IO]
```

### Deriving `Gen` and others

Note at the time of writing `magnolify-scalacheck` [does not yet work with Scala 3](https://github.com/spotify/magnolify/pull/857). You might have to figure out your own way to derive `scalacheck` instances, or write them out manually. I don't currently have a solution to offer you.

# Logging

It's best to use a logging library which integrates with your effect system, and for sure you want to avoid ones that are bloated to the point they cause [severe security issues](https://en.wikipedia.org/wiki/Log4Shell).

[`valskalla/odin`](https://github.com/valskalla/odin) used to be a great alternative, but it's no longer maintained and has no Scala 3 version, so use the maintained fork [scalafreaks/odin](https://github.com/scalafreaks/odin) instead.

`odin` is purely functional, integrated with `cats-effect` and can provide principled MDC context propagation via `ReaderT`, if you need that. In addition, it's not just a purely functional wrapper, but has an `slf4j` implementation, which foregoes the need for `logback` or anything else.

As good practice dictates, I recommend to log to console only, and if logs need to go anywhere else, let external infrastructure handle that.

```scala
package yourproject

import cats.effect.*
import io.odin.*

package object log:
  def defaultLogger[F[_]: Sync]: Logger[F] = consoleLogger[F](minLevel = Level.Info)
```

An alternative to `odin` is [`woof`](https://github.com/LEGO/woof). The main difference is `woof` currently uses `IOLocal` for context propagation, whereas `odin` allows for a `ReaderT` / `Ask` implementation. If you don't have strong opinions about this, or if you have no need for context propagation, they can be viewed as mostly similar.

# Observability / distributed tracing

Observability means that while executing our program in production, we collect and persist data about the execution in a structured manner, so that we can answer questions about how the program is performing when things go awry.

We're able to answer questions such as

- What was the user id of that request that ended with a 5xx
- Where was time spent for the 1% slowest requests in the last hour
- What's the latency distribution of X in this deployment versus the previous deployment

, without having to resort to hopeless log grepping and `println`s.

If you're not already doing that, you should! It makes your life operating software in production a lot easier, and you end up with a better overall product.

My current library of choice is [natchez](`https://github.com/typelevel/natchez`), because that's what my database library of choice currently uses. As an actual observability backend, I default to [Honeycomb](https://www.honeycomb.io/). No strong opinions here - I just find it pretty easy to work with, and the pricing feels reasonable. `natchez` has backend implementations for other popular platforms, including Datadog and AWS X-Ray.

Library-wise, the landscape has historically been a bit messy / fractured. I think the community might eventually settle on `https://github.com/typelevel/otel4s` - I see that my database library has switched to that on its main, unreleased branch.

# Refinement types

## Primer 
Refinement types are types with a predicate, such as (pseudocode)

- `type PosInt = x: Int | x > 0`
- `type ValidPort = x: Int | x >= 0 & x <= 65535`
- `type Email = x: String | x matches ValidEmailRegex` 

, and so forth.

With refinement types, when using static data (literals), we're able to accept or reject a value at compile-time. Example (pseudocode): 

```scala
  val x: PosInt = 23 // compiles
  val x: PosInt = -1 // compile-time error: -1 is not a PosInt since predicate GreaterThan(0) does not hold 
```

When we're dealing with dynamic data which might not fulfil the predicate, the possibility of error is reflected in the type:

```scala
  val x: Either[String, PosInt] = refine[PosInt](21)
```
In any case, invalid states are made unpresentable, which lets us model domain data types more precisely, and catch data errors earlier, in the outermost layers of our program.
 
## Libraries

Under scala 2.x, the go-to library for refinement types was [refined](https://github.com/fthomas/refined). Unfortunately, while the library is cross-published, compile-time refinement of literals with Scala 3 is [not possible](https://github.com/fthomas/refined/issues/932).

The solution is to switch to [iron](https://github.com/Iltotore/iron), which provides both compile-time and runtime refinement but **only exists for Scala 3**.

I believe this state of affairs makes the situation

- OK for greenfield projects
- Annoying for codebases migrating from Scala 2 to Scala 3, since switching out such a fundamental library while also switching the language version is a pretty big-bang refactoring
- Very unfortunate for large codebases migrating from Scala 2 to Scala 3, for the same reasons aggravated by scale
- I expect, extremely complicated for libraries which use refinement types and must be cross-published to Scala 2 and Scala 3

There's good news though! `iron` is superior to `refined` in that refined types are subtypes of their base types, e.g. `Int :| Positive` is a subtype of `Int` (whereas `refined` uses wrapper types to represent refined types).

This, along with the fact that conversions between the refined type and its base type are `inline`d, gives the capability, in many situations, to have zero runtime overhead when using refinement types.

In addition, `iron` comes with integrations for your favourite libraries that you've come to love and expect, such as `circe`, `ciris`, `skunk`, `doobie`, `cats`, `tapir`, etc.

# Typeclass derivation

, which is the compiler being able to infer typeclass instances that are "obvious", instead of us writing them out by hand.

This is by no means critical, since you can always write your own instances, but it's handy and makes programs more terse. It's also good practice as a programmer to automate what you can, because sometimes the most hard to diagnose mistakes are found in boring, seemingly hard to get wrong code.

The situation here has historically been somewhat messy because of the matrix of
- Multiple base mechanisms to implement typeclass derivation (chiefly `shapeless` and `magnolia`)
- Multiple flavours / ways of typeclass derivation used by the target libraries that you want to derive instances for ("semi-auto", "full-auto", etc)

Out of the mechanisms, `magnolia` is preferred over `shapeless` in Scala 2, because of the superior (compile-time) performance.

I believe the situation might have gotten even more colourful with Scala 3, as exemplified by this line of documentation in the `typelevel/kittens` project:

```
There are five options for type class derivation on Scala 3.
```

Rightly so, because Scala 3 provides a basic typeclass derivation mechanism itself, and syntax sugar for typeclass derivation at the call site.

Below follow options for specific libraries. I haven't yet had a chance to test these out on large-scale codebases, so your mileage (and compile times) may vary.


## `circe` instances

Use Scala `derives` clause, which is going to use Scala 3 metaprogramming facilities under the hood.

```scala
package yourproject

import io.circe.Codec

final case class HealthResponse(
    message: String
) derives Codec
```

## `tapir` instances

Use Scala `derives` clause, which is going to use `magnolia` facilities under the hood, as far as I can tell.

```scala
package yourproject

import sttp.tapir.Schema

final case class Foo(
    bar: String
) derives Schema
```

## `cats` instances

Use [`kittens`](https://github.com/typelevel/kittens), which uses `shapeless3` under the hood. `kittens` supports Scala 3 `derives` clause - see their docs.

[`magnolify/cats`](https://github.com/spotify/magnolify/tree/main/cats/src) used to be an option, but at the time of writing it doesn't support Scala 3.

## `scalacheck` instances

As already pointed out, there's no good solution currently. [`magnolify/scalacheck`](https://github.com/spotify/magnolify/tree/main/scalacheck/src) used to be an option, but at the time of writing it doesn't support Scala 3. You might attempt to implement derivation via `magnolia` on your own, or else revert to writing instances manually.

# Configuration management

# Database connectivity

# Database migrations

# Cryptography

# HTTP endpoint documentation

# Others 

# Further work