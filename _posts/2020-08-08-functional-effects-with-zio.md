---
title: "Functional Effects with ZIO"
date: 2020-08-08
header:
  image: "/images/zio-header.jpg"
tags: [scala, jvm, zio, fp]
excerpt: "Scala, JVM, ZIO, Functional Programming"
---

{: .text-left}
[ZIO 1.0](https://www.linkedin.com/pulse/zio-10-released-john-de-goes/) is finally released after three years of active development and more than 300 contributors, so I decided to write a practical introduction to ZIO and functional effects.

In my [last article]({% post_url 2020-05-24-a-taste-of-functional-java %}), I was talking about some basic concepts related to functional programming.
We already know that Java is not a good fit for purely functional programming, and there are no libraries that can help you with that, but you can use [Vavr](https://www.vavr.io/) to make your codebase more functional. This style is often referenced as object-functional programming. 

As a result of the extremely slow evolution of Java, we got modernized alternatives, which are also more suitable for functional programming. One of them is Kotlin with [Arrow](https://arrow-kt.io/) library which seems good to me, but Scala is the most popular and most mature choice for functional programming on the JVM with a few high-quality libraries.
I'm recommending you to read [this article](https://scalac.io/why-does-scala-win-against-kotlin-senior-engineers-opinion/) if you are interested in the comparison between Scala and Kotlin.

[ZIO](https://zio.dev/), [Cats Effect](https://typelevel.org/cats-effect/), and [Monix](https://monix.io/) are top three functional effects systems for Scala. Of course, a [Scalaz](https://scalaz.github.io/7/) was an inspiration for these libraries, but I feel that its popularity is declining. This article will explain what functional effects are and the benefits of using them. I will be using Scala with [ZIO](https://zio.dev/).

## ZIO In a Nutshell

ZIO is a Scala library that provides many features for developing concurrent, parallel, asynchronous, and resources safe code in a purely functional and more straightforward way. ZIO can provide everything necessary for building any modern, highly concurrent application.

The core type in the ZIO library is called `ZIO`, and it has three type parameters.
Here is an example of a pretty simplified model of the ZIO effect type. An actual implementation is much more complex.

```scala
final case class ZIO[-R, +E, +A](run: R => Either[E, A])
```

The ```ZIO``` type is just a case class that contains a function.
The function takes a value of type `R` and produces either an error on the left-hand side or success on the right-hand side of types `E` and `A`.

Let's see what is ZIO really about and demystify the type parameters:
* **R** - Environment Type. The effect may need some dependency of type `R` to be executed. If this type parameter is `Any`, the effect has no environmental requirements.
* **E** - Failure Type. The effect may fail with some value of type `E`. If this type parameter is `Nothing`, it means the effect cannot fail because there are no values of type `Nothing`.
* **A** - Success Type. The effect may succeed with a value of type `A`. If this type parameter is `Unit`, it means the effect produces no useful information, while if it is `Nothing`, it means the effect runs forever (or until failure).

Type parameters explained again, through example.

```scala
val effect: ZIO[Connection, UserNotFoundError, User] = ???
```
We defined ```effect```, a ZIO effect that reads something from the database. To run the effect will need ```Connection```. It can fail with ```UserNotFoundError```and succeed with ```User```. Note that ZIO effects can fail with any subtype of `Throwable` or any user-defined ADT, depending on the programmer's choice of representing failures in their applications.

## Effects are workflows

Effects are immutable values, similar to Scala's immutable types like `Option`, `Either`, `Try`, `List`, etc.
Methods on these types give you a new value, the result of applying a specific operation to it. An example is mapping over `Option` or `List`, which will give you back a new `Option` or `List`. Likewise, every operation on a ZIO effect will produce a new effect resulting from a specified operation against the original effect.

An effect is a workflow that can represent a sequential, asynchronous, concurrent, or parallel computation that will allocate resources and safely free them after use. Effects are purely descriptive and lazy descriptions of workflows.
Effects are composable, which means that we can combine them in numerous ways, using methods defined on them, and as a result of composition, we will get a completely new effect. Effects are used to model some common operations or sequence of operations, like database interaction, RPC calls, WebSocket connections, etc. Besides all of that, we can use effects to model failures, recovery, scheduling, resource management (allocation and deallocation), etc.


## Simplified ZIO Implementation

First, let's define a ```ZIO``` type with a few basic combinators.

```scala
final case class ZIO[-R, +E, +A](run: R => Either[E, A]) { self =>
    def map[B](f: A => B): ZIO[R, E, B] = ???

    def flatMap[R1 <: R, E1 >: E, B]
        (f: A => ZIO[R1, E1, B]): ZIO[R1, E1, B] = ???

    def zip[R1 <: R, E1 >: E, B]
        (that: ZIO[R1, E1, B]): ZIO[R1, E1, (A, B)] = ???

    def either: ZIO[R, Nothing, Either[E, A]] = ???

    def provide(r: R): ZIO[Any, E, A] = ???

    def orDie(implicit ev: E <:< Throwable): ZIO[R, Nothing, A] = ???
}
```

A few helper methods are defined in ZIO companion objects to construct effects.

```scala
object ZIO {
    def succeed[A](a: => A): ZIO[Any, Nothing, A] = ???

    def fail[E](e: => E): ZIO[Any, E, Nothing] = ???

    def effect[A](sideEffect: => A): ZIO[Any, Throwable, A] = ???

    def environment[R]: ZIO[R, Nothing, R] = ???

    def access[R, A](f: R => A): ZIO[R, Nothing, A] = ???

    def accessM[R, E, A](f: R => ZIO[R, E, A]): ZIO[R, E, A] = ???
}
```

Now, let's implement our helper methods.

From method definition, you can see that the `succeed` effect that we are building doesn't require anything, it can't fail, and it will succeed with a value of type `A`.

```scala
def succeed[A](a: => A): ZIO[Any, Nothing, A] =
    ZIO(_ => Right(a))
```

To construct the effect, we need to use the ZIO effect constructor, which requires the function: `R => Either[E, A]`.
Since our effect doesn't have any environment dependencies, in a ZIO constructor, we ignore environment type and use the `Right` constructor to lift our value of type `A` into `Either`, which is the required output type.


```scala
def fail[E](e: => E): ZIO[Any, E, Nothing] =
    ZIO(_ => Left(e))
```

Similarly, the `fail` effect is constructed. It's an effect that doesn't require anything, can't succeed, but can fail with the value of type `E`. So, it's just the opposite of our `succeed` effect, and implementation is similar. Using the ' Left ' constructor, we are lifting our error into `Either`.

If you are not familiar with `Either`, it is a type that comes from the Scala standard library, and it represents a value of one of two possible types (a disjoint union). Instances of `Either` are either an instance of `Left` or `Right`, representing failure and success, respectively.

After implementing pretty simple effect constructors, we encountered the `effect` method, which is more interesting than the previous ones.
This method captures a code that performs side effects (interaction with the file system, database, web server, etc.) and defers its execution. The problem with side-effects is that they are doing not describing, and in functional programming, we want to defer interaction with the real world as long as possible. We use [by-name parameters](https://docs.scala-lang.org/tour/by-name-parameters.html), which means that what is in the parameter list won't be evaluated immediately, it will be stored into something that is called a thunk, and it will be evaluated much later when the effect is executed. Thunk is a pointer to a function that executes some code, and it's a way to make the execution of some code lazy. This technique allows us to take some side-effectful code eager and turn it into a lazy description.

```scala
def effect[A](sideEffect: => A): ZIO[Any, Throwable, A] =
    ZIO(_ => Try(sideEffect).toEither)
```
From the method definition, you can see that the `effect` method, which converts eager code to a lazy description, doesn't require anything.
As an argument, we have a piece of code that performs a side effect. Side-effect code may throw an exception, and we want to translate it into a failure value. The implementation of `effect` is simple: we will ignore the environment, since we don't need any, and try to execute a thunk that is passed as a method argument and translate it into failure or success.

Here we have an `environment` method that builds an effect that requires `R` and succeeds with a value of type `R`. This method describes an effect that will need an `R` to be executed and introduce the concrete type for `R`.

```scala
def environment[R]: ZIO[R, Nothing, R] = 
    ZIO(r => Right(r))
```

`access` is a method that will allow us to make the environment of the type `R` using a provided function `f` and project some piece of information from that of type `A`.

```scala
def access[R, A](f: R => A): ZIO[R, Nothing, A] = 
    ZIO(r => Right(f(r)))
```
To implement the `access` method, we are going to use `r`, which represents our environment, feed that `r` into our function `f` to get `A` and lift that result into `Either`, using `Right` constructor, since this effect can't fail.

The final method in our companion object is `accessM`, which represents an effectful version of the `access` method. You noticed the `M` suffix, which identifies an effectful version of some method. This means that any method with the suffix `M` will accept a function that returns effects instead of a plain value.

```scala
def accessM[R, E, A](f: R => ZIO[R, E, A]): ZIO[R, E, A] = 
    ZIO(r => f(r).run(r))
```

Implementation can be tricky, but try to follow the types, and there shouldn't be any problems.

We've finished the implementation of methods that helps us to build a wide range of effects. The next step is to implement basic combinators for our `ZIO` effect type, which will allow us to combine and transform our effects.

The most common combinator in functional libraries is a `map`, of course. It receives a function that transforms our success value of type `A` to the success value of type `B`. So for a given function that turns `A` into `B`, we need to take the current ZIO effect and return a new ZIO effect with the success type transformed to `B`.

```scala
def map[B](f: A => B): ZIO[R, E, B] = 
    ZIO(r => self.run(r).map(f))
```

After running the current ZIO effect with `r`, we are getting `Either[E, A]` as a result. Then we are mapping over `Either` with a given function `f: A => B`. The result is a new effect that can succeed with a value of type `B`.

The following combinator is the famous `flatMap`, which allows us to combine context-sensitive, sequential operations. The method definition looks scary, but it essentially receives a function that will lift the plain value of type `A` into a ZIO effect.

```scala
def flatMap[R1 <: R, E1 >: E, B]
    (f: A => ZIO[R1, E1, B]): ZIO[R1, E1, B] = 
        ZIO(r1 => self.run(r1).flatMap(a => f(a).run(r1)))
```

To get `ZIO[R1, E1, B]`, we are running the current `ZIO` effect with `r1`, which will give us `Either[E, A]`. Then we are flat mapping over `Either` to get `A` out of it. When we finally have `a` of type `A`, we are feeding that into our function `f` that will give us back `ZIO[R1, E1, B]`. We need `Either[E1, B]`, so we will just run the result of our function `f`, since it is the ZIO effect, to get what we need.

The following combinator is `zip`, which combines two effects to get a new effect. It can be expressed via `flatMap` and `map` that we already implemented.

```scala
def zip[R1 <: R, E1 >: E, B](that: ZIO[R1, E1, B]): ZIO[R1, E1, (A, B)] =
    self.flatMap(a => that.map(b => a -> b))
```

You can use the `either` combinator to move the error channel into the success channel in the form of the `Either`.
We are not removing error with this combinator, and it will be just translated into a success channel.

```scala
def either: ZIO[R, Nothing, Either[E, A]] =
    ZIO(r => Right(self.run(r)))
```

As our effect representing the output of `either` can't fail, we are just running our current `ZIO` effect and lifting the result into `Either` using the `Right` constructor.

The following combinator, `provide`, removes the environment dependency from the current effect.

```scala
def provide(r: R): ZIO[Any, E, A] =
    ZIO(_ => self.run(r))
```

We need to return the `ZIO` effect that doesn't require an environment. So we are just running our current effect with the environment of type `R` given as a method argument. We have managed to do this here because we just transformed the effect that needed an `R` into an effect that doesn't require anything to be executed.

The last combinator is called `orDie`. This strange-looking definition says that this method can be called on the effect that can fail only with some subtype of `Throwable`.

```scala
def orDie(implicit ev: E <:< Throwable): ZIO[R, Nothing, A] =
    ZIO(r => self.run(r).fold(throw _, Right(_)))
```

This combinator removes the error parameter in the following way:

* Effect is run with `r` and result will be `Either[E, A]`
* After that, we are folding over `Either`
* If the effect fails, it will throw an exception
* If not, we will wrap the success value into `Right`

This combinator is useful when there is a possibility of fatal errors that we cannot recover from.

We implemented our primitive version of the `ZIO` effect type. Let's write some simple console application that will ask the user for a name and print it out.

## At World's End

First, we need to define an effect that will print some text on the console.

```scala
def putStrLn(line: String): ZIO[Any, Nothing, Unit] =
    ZIO.effect(println(line)).orDie
```

The following effect that we will need is printing out the name that the user entered.

```scala
val readLine: ZIO[Any, Nothing, String] =
    ZIO.effect(scala.io.StdIn.readLine()).orDie
```

You noticed that we create two effects that can't fail using the `effect` constructor and `orDie` combinator. When we have defined all the effects needed for our program, we need to combine them and finally write a function responsible for executing our final effect that represents our whole application.

```scala
val program: ZIO[Any, Throwable, Unit] =
  putStrLn("Hello, what is your name?").flatMap { _ =>
    readLine.flatMap(name => putStrLn(s"Your name is: $name"))
  }
```

As we mentioned earlier, effects are just immutable values describing workflows, and they don't do anything. In functional programming, all problems are solved using values.

Finally, we need to create a method that will take the `ZIO` effect type and execute it to get a plain value.

```scala
def unsafeRun[A](zio: ZIO[Any, Throwable, A]): A =
    zio.run(()).fold(throw _, a => a)
```

Our effect doesn't require anything to run, so we pass a `Unit` value to the `run` method. We are getting `Either[Throwable, A]`, and after folding over it,  we finally have to deal with the exception. If we get an instance of `Throwable`, we will throw it, and if we get the success value of type `A`, we will return it.

Our `main` method will look like this:

```scala
def main(args: Array[String]): Unit =
    unsafeRun(program)
```
We started by defining multiple effects. After that, we combined them and ended up with one final effect. If we want to run an effect, we need to translate the description of a workflow into actual actions described using a method called `unsafeRun`. Usage of prefix `unsafe` means that this is no longer functional programming. It indicates that we are on edge between purely functional programming and procedural programming. `unsafeRun` function or method is not a function in the sense of functional programming since it performs a side-effect, may not be deterministic and can throw an exception if executed. Using `unsafeRun` represents the final step before the famous end of the (*functional*) world.
It’s good practice to use this function as little as possible. Ideally, it would be only once.


## Complete code

```scala
final case class ZIO[-R, +E, +A](run: R => Either[E, A]) { self =>
    def map[B](f: A => B): ZIO[R, E, B] =
        ZIO(r => self.run(r).map(f))

    def flatMap[R1 <: R, E1 >: E, B]
        (f: A => ZIO[R1, E1, B]): ZIO[R1, E1, B] = 
        ZIO(r1 => self.run(r1).flatMap(a => f(a).run(r1)))

    def zip[R1 <: R, E1 >: E, B]
        (that: ZIO[R1, E1, B]): ZIO[R1, E1, (A, B)] = 
        self.flatMap(a => that.map(b => a -> b))

    def either: ZIO[R, Nothing, Either[E, A]] = 
        ZIO(r => Right(self.run(r)))

    def provide(r: R): ZIO[Any, E, A] =
        ZIO(_ => self.run(r))

    def orDie(implicit ev: E <:< Throwable): ZIO[R, Nothing, A] =
        ZIO(r => self.run(r).fold(throw _, Right(_)))
}

object ZIO {
    def succeed[A](a: => A): ZIO[Any, Nothing, A] =
        ZIO(_ => Right(a))

    def fail[E](e: => E): ZIO[Any, E, Nothing] =
        ZIO(_ => Left(e))

    def effect[A](sideEffect: => A): ZIO[Any, Throwable, A] =
        ZIO(_ => Try(sideEffect).toEither)

    def environment[R]: ZIO[R, Nothing, R] =
        ZIO(r => Right(r))

    def access[R, A](f: R => A): ZIO[R, Nothing, A] =
        ZIO(r => Right(f(r)))

    def accessM[R, E, A](f: R => ZIO[R, E, A]): ZIO[R, E, A] =
        ZIO(r => f(r).run(r))
}

object App {
    def putStrLn(line: String): ZIO[Any, Nothing, Unit] =
        ZIO.effect(println(line)).orDie

    val readLine: ZIO[Any, Nothing, String] =
        ZIO.effect(scala.io.StdIn.readLine()).orDie

    val program: ZIO[Any, Throwable, Unit] =
        putStrLn("Hellow, what is your name?").flatMap { _ =>
            readLine.flatMap(name => putStrLn(s"Your name is: $name"))
        }

    def unsafeRun[A](zio: ZIO[Any, Throwable, A]): A =
        zio.run(()).fold(throw _, a => a)

    def main(args: Array[String]): Unit =
        unsafeRun(program)
}
```

## Summary

We reviewed some of the most popular options for functional programming on JVM, especially in the Scala ecosystem, since it's the most powerful language on JVM for functional programming. After that, we introduced ZIO and explained what functional effects are by implementing a simplified version of ZIO. In some of the following blog posts, the focus will be on ZIO and solving real-world problems. If you are getting started and want to know more about ZIO, you can start reading [A Brief History of ZIO](https://degoes.net/articles/zio-history).

The motivation for writing this blog post comes from [Functional Effects Training by Ziverge](https://ziverge.com/training/fs-effects), and I recommend it to everyone serious about learning more about functional programming and ZIO especially.
[ZIO community](https://github.com/zio) is very welcoming, so feel free to join on [Discord channel](https://discord.com/invite/2ccFBr4) if you are interested in it, or even want to start contributing.

You can find me at:
* [Linkedin](https://www.linkedin.com/in/aleksandar-skrbic/)
* [Github](https://github.com/aleksandarskrbic)

Or send me a question to [skrbic.alexa@gmail.com](skrbic.alexa@gmail.com)

## Credits

Thanks to [@nathanknox](https://github.com/nathanknox) for proofreading and all ZIO contributors!