# Monad Transformer Magic

Say we have a function that computes an Int, but could fail:

```scala
def maybeInt(x: Int): Either[Throwable, Int] =
  Right(x)
// defined function maybeInt
```

And we want to use it to get some values in one go for later use in a fail-fast manner:

```scala
val myInts = for {
  x <- maybeInt(1)
  y <- maybeInt(2)
  z <- maybeInt(3)
} yield (x, y, z)
// myInts: Either[Throwable, (Int, Int, Int)] = Right((1, 2, 3))
```

We can do something with them now by pattern matching:

```scala
myInts match {
  case Left(error) => println("Oh noes!")
  case Right((x, y, z)) => println(s"x=$x y=$y z=$z")
}
// x=1 y=2 z=3
```

This is all well and good, but we've performed an eager evaluation here and caused some side effects.
Let's fix that by wrapping things in the `IO[_]` monad.

We'll also pretend our function performs some side effects in order to compute our `maybeInt` value.
We don't want to eagerly evaluate these now, so we'll wrap in an `IO`.

First we need to import the IO monad from the Cats Effect library:

```scala
import cats.effect.IO
//  import cats.effect.IO
```

You can find out more about the Cats Effect `IO` monad [in their docs](https://typelevel.org/cats-effect/docs/2.x/datatypes/io).

```scala
def maybeIntPure(x: Int): IO[Either[Throwable, Int]] = {
  IO {
    // Pretend we do some side effects here - we need to wrap in IO
    Right(x)
  }
}
// defined function maybeIntPure
```

Okay great, all is well, let's try using it as we did before.

```scala
val myInts = for {
  x <- maybeIntPure(1)
  y <- maybeIntPure(2)
  z <- maybeIntPure(3)
} yield (x, y, z)
// myInts: IO[(Either[Throwable, Int], Either[Throwable, Int], Either[Throwable, Int])] = Bind(
//   Delay(ammonite.$sess.cmd10$$$Lambda$1639/0x00000008408ea840@14e93c46),
```

Eek, that's not a very helpful type we've got back. We've lost our fast-fail ability that means we only get one `Either[...]` back at the end. This would be a pain to pattern match against!

Okay, so let's try using the [tupled](https://www.scalawithcats.com/dist/scala-with-cats.html#apply-syntax) function from [Semigroupal](https://www.scalawithcats.com/dist/scala-with-cats.html#sec:applicatives) to combine our result to only have one `Either[...]`.
This comes from the [Cats library](https://typelevel.org/cats/resources_for_learners.html).

We'll need to do some imports first:

```scala
import cats.implicits._
// import cats.implicits._
```

```scala
val myInts = for {
  x <- maybeIntPure(1)
  y <- maybeIntPure(2)
  z <- maybeIntPure(3)
} yield (x, y, z).tupled
// myInts: IO[Either[Throwable, (Int, Int, Int)]] = Bind(
//   Delay(ammonite.$sess.cmd0$$$Lambda$1466/0x0000000840867040@56ccd751),
//   ...
```

Well that looks better. Let's use our values to do something:

```scala
myInts.flatMap {
  case Left(error) => IO(println("Oh noes!"))
  case Right((x, y, z)) => IO(println(s"x=$x y=$y z=$z"))
}
.unsafeRunSync() // Evaluating here just for example
// x=1 y=2 z=3
```

Still looking good, so let's try this technique with a slightly more complicated example. Now the value for `y` requires the result of computing `x` and `z` requires `y`.

```scala
val myInts = for {
  x <- maybeIntPure(1)
  y <- maybeIntPure(x)
  z <- maybeIntPure(y)
} yield (x, y, z).tupled
// cmd2.sc:3: type mismatch;
//  found   : Either[Throwable,Int]
//  required: Int
//     y <- maybeInt(x)
//                   ^
// Compilation Failed
```

Oh dear.

Because the return type of `maybeIntPure` is `IO[Either[Throwable, Int]]`, the result of the `flatMap`s that happen in the for comprehension are `Either[Throwable, Int]`.
In other words, the `x` in `x <- maybeIntPure(1)` is of type `Either[Throwable, Int]`, not `Int`.

This is bad because it means we can no longer pass `x` into the next function to get `y`.  
So what to do? - Enter monad transformers!

We're going to be using a monad transformer from the Cats library, so let's import it:

```scala
import cats.data.EitherT
// import cats.data.EitherT
```

Let's use it to wrap our `Either[Throwable, Int]` from before:

```scala
val myIOMaybeInt = EitherT(maybeIntPure(1))
// myIOMaybeInt: EitherT[IO, Throwable, Int] = EitherT(Delay(...))
```

So why does this matter, and what have we actually done?

```scala
val myInts = (for {
  x <- EitherT(maybeIntPure(1))
  y <- EitherT(maybeIntPure(x))
  z <- EitherT(maybeIntPure(y))
} yield (x, y, z)).value
// myInts: IO[Either[Throwable, (Int, Int, Int)]] = Bind(
//   Delay(ammonite.$sess.cmd0$$$Lambda$1466/0x0000000840867040@3dc95b8b),
```

Using the `EitherT[...]` monad transformer we're combining the `flatMap` functions of both the `IO` and `Either` types in our `IO[Either[Throwable, Int]]`. This means when we call `flatMap` on our `EitherT` transformed value we get the innermost wrapped value (the `Int`), rather than the outer wrapped value like we were before (the `Either[Throwable, Int]`).

We've managed to keep our functions effect free, and preserved our fast fail ability. Excellent!

We could now use our values as we did before:

```scala
myInts.flatMap {
  case Left(error) => IO(println("Oh noes!"))
  case Right((x, y, z)) => IO(println(s"x=$x y=$y z=$z"))
}
.unsafeRunSync() // Evaluating here just for example
// x=1 y=1 z=1
```

You can find out more about Monad Transformers in [Scala with Cats](https://www.scalawithcats.com/dist/scala-with-cats.html#sec:monad-transformers).
