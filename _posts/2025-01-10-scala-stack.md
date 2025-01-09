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

# Language version

Prior to this exercise I hand't done any Scala 3, personally or professionally. I've been reading and writing code for a while, so I figured picking up the new language version itself wouldn't be too much of a hassle. I was more worried whether all my libraries of choice would work with Scala 3, and in 2025 they all do, with a couple of exceptions I'll note in subsequent paragraphs. Those have adequate replacements, so for new **services**, I'd recommend defaulting to Scala 3.

## Language changes 

I didn't find picking up language changes hard. My current personal and work projects try to stick to a relatively small subset of Scala that's purely functional, and all compile with `-Xsource:3`, so the transition was mostly painless. Here's what I gradually did while building my new project.

- Pick up new control flow syntax - not mandatory (with Scala `3.3.4`) and can be automated, as described later. This mostly boiled down to writing `if a then foo else bar` in place of `if (a) foo else bar`
- Stop writing optional braces - not mandatory and can be automated, as described later.
- Use `given / using` in place of `implicit`
- Use `extension` in place of `implicit class`
- Use `enum` in place of `sealed trait` whenever describing sum types. If my understanding is correct, this mostly works, however they currently don't have 100% feature parity. Specifically enums cannot nest enums. You can revert to `sealed trait` whenever facing issues, granted that's not great for consistency.

## Cross-building own libraries

You might be in the position of maintaining libraries that need to be used across Scala 2 and Scala 3 services.

One way to solve this, if it works for you, is to code your libraries to a subset of Scala 2 that's also valid Scala 3. For me, this boiled down to `"-Xsource:3-cross"`, or, more completely:

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

In addition I use a micro-library I'm the author of, [weaver-test-extra](https://github.com/dimitarg/weaver-test-extra), to aid in purely functional test setup and teardown. (See the repo readme for reasoning, use and limitations).

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

[`valskalla/odin`](https://github.com/valskalla/odin) used to be a great alternative, but it's no longer maintained and has no Scala 3 version, so use[scalafreaks/odin](https://github.com/scalafreaks/odin) instead.

`odin` is purely functional, integrated with `cats-effect` and can provide principled MDC context propagation via `ReaderT`, if you need that. In addition, it's not just a purely functional wrapper, but has an `slf4j` implementation, which foregoes the need for `logback` or anything else.

As good practice dictates, log to console only, and if logs need to go anywhere else, let external infrastructure handle that.

```scala
package yourproject

import cats.effect.*
import io.odin.*

package object log:
  def defaultLogger[F[_]: Sync]: Logger[F] = consoleLogger[F](minLevel = Level.Info)
```

An alternative to `odin` is [`woof`](https://github.com/LEGO/woof). The main difference is `woof` uses `IOLocal` [for context propagation](https://github.com/LEGO/woof/issues/130#issue-1573872587), whereas `odin` uses `ReaderT`. If you don't have [strong opinions](https://github.com/LEGO/woof/issues/130#issuecomment-2462239678) about this, or if you have no need for context propagation, either alternative is fine.

# Observability / distributed tracing

I find it's sloppy to write programs where the answer to "what went wrong" is log grepping, and the answer to "why is this so slow" is "I have no idea, let's do some log grepping and print out a bunch of `System.currentTimeMillis()` in addition".

My current library of choice is [natchez](`https://github.com/typelevel/natchez`), because that's what my database library of choice currently uses. As an actual observability backend, I default to [Honeycomb](https://www.honeycomb.io/). No strong opinions here - I just find it pretty easy to work with, and the pricing feels reasonable.

Library-wise, the landscape is a bit messy / fractured today. I think the community might eventually settle on `https://github.com/typelevel/otel4s` - I see that my database library has switched to that on its main, unreleased branch.

# Optics
TODO
