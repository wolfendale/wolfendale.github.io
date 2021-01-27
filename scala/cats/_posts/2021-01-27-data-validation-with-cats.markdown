---
title: Data validation with cats
---

{:.lead}
`Validated[A, B]` is a data type provided by cats. It's like an `Either[A, B]` which is able to collect errors on the left hand side.

This is _super_ useful!

```scala
def program(data: Map[String, Int]): Validated[List[String], Int] = {

  def get(key: String): Validated[List[String], Int] =
    data.get(key).toValid(List(s"$key is missing!"))

  (get("one"), get("two")).mapN(_ + _)
}

program(Map("one" -> 1, "two" -> 2)) // Valid(3)
program(Map("one" -> 1)) // Invalid(List(two is missing!))
program(Map("two" -> 2)) // Invalid(List(one is missing!))
program(Map.empty) // Invalid(List(one is missing!, two is missing!))
```

This is great for composing programs that you want to collate errors for, but it only works for independent values. If we need to use the inner value of a `Validated` to work out what to do _next_ there's a caveat.

```scala
def program(data: Map[String, Int]): Validated[List[String], Int] = {

  def get(key: String): Validated[List[String], Int] =
    data.get(key).toValid(List(s"$key is missing!"))

  get("first").andThen {
    case 1 => get("one")
    case 2 => get("two")
    case _ => List("first is not in the map!").invalid
  }
}

program(Map("first" -> 1, "one" -> 1, "two" -> 2)) // Valid(1)
program(Map("first" -> 2, "one" -> 1, "two" -> 2)) // Valid(2)
program(Map("first" -> 1)) // Invalid(List(one is missing!))
program(Map("first" -> 2)) // Invalid(List(two is missing!))
program(Map.empty) // Invalid(List(first is missing!))
program(Map("first" -> 1337)) // Invalid(List(first is not in the map!))
```

Here you can see that if we cannot get a value for `"first"` then we don't get the error message for the next value. This makes sense - how would we know what error message to use? We don't know if we were going to look for `"one"` or `"two"`.

The other thing that you might think, is that `andThen` looks _a lot like_ `flatMap`.

They look the same but they're _subtly_ different. Because `Validated` collates errors on the left hand side, it breaks the Monad laws so it would be incorrect to have a `flatMap` method for `Validated`. The lack of `flatMap` can be frustrating as it means that we cannot use `Validated` instances
in a `for` comprehension.
 
When values are independent it makes sense that programs that operate on them could be composed in _parallel_. However, programs operating on dependent values must be composed in _sequence_.

Using the examples above:

- If we want to get `"one"` and `"two"` they are independent, we can fetch them separately and then combine them together once we have both.

- If we want to decide which value we want to retrieve, we cannot fetch `"one"` or `"two"` until we have fetched `"first"`.

I previously mentioned that `Validated` and `Either` are almost the same. In fact they are _so_ similar we can think of them as two sides of the same coin. Cats defines the `Parallel` type class to describe the relationship between types with the same structure but where one composes in parallel and the other composes in sequence. This gives us the ability to convert values between an equivalent _sequential_ type and _parallel_ type.

Usually you don't need to use `Parallel` directly, instead it provides extra syntax for instances of any _sequential_ type which has a `Parallel` instance pairing it with a _parallel_ type. This lets you decide how you want to compose operations on data of this type.

```scala
def sequentialProgram(data: Map[String, Int]): Either[List[String], Int] = {

  def get(key: String): Either[List[String], Int] =
    data.get(key).toRight(List(s"$key is missing!"))

  (get("one"), get("two")).mapN(_ + _)
}

def parallelProgram(data: Map[String, Int]): Either[List[String], Int] = {

  def get(key: String): Either[List[String], Int] =
    data.get(key).toRight(List(s"$key is missing!"))

  (get("one"), get("two")).parMapN(_ + _)
}

sequentialProgram(Map.empty) // Left(List(one is missing!))
parallelProgram(Map.empty) // Left(List(one is missing!, two is missing!))
```

> Note: Our programs now return an `Either[List[String], Int]` instead of `Validated[List[String],Int]`.

**This is even better**! We aren't even using
`Validated` but we _still get parallel composition!_

This is fundamentally more versatile than using `Validated` directly, and in my experience, the code produced is a lot easier to understand. It just comes down to a few extra methods on collections of `Either`s.

Generally, I find this is the best way to use `Validated`, to not use it! Or at least, to use it via `Parallel` so you can have your cake and eat it too.

For more information on `Parallel` see the [cats documentation](https://typelevel.org/cats/typeclasses/parallel.html)

## Extra credit

### Choosing an error type

In the above examples, we used a `List[String]` as our error type. There are downsides to this though:

Firstly, there is an instance that can be created that doesn't make much sense: `Left(List())` would represent an invalid computation with no error. In the programs above this is impossible, but we can do better and prove to the compiler that this is the case by substituting our `List` with a `NonEmptyList`. This has the added benefit of showing other developers working on our code that they don't need to consider this case.

Secondly, errors are _appended_ to our collection when we run the program and appending to a `List` is pretty inefficient. Thankfully, Cats includes its own data type for use in situations like this: `Chain` and its non-empty equivalent `NonEmptyChain`. These data types can largely be used as drop in replacements for `List` and `NonEmptyList` and have constant time append and prepend. The [cats documentation](https://typelevel.org/cats/datatypes/chain.html) includes benchmark results for a variety of operations against different collections.

```scala
def program(data: Map[String, Int]): EitherNec[String, Int] = {

  def get(key: String): EitherNec[String, Int] =
    data.get(key).toRightNec(s"$key is missing!")

  (get("one"), get("two")).parMapN(_ + _)
}

program(Map.empty) // Left(Chain(one is missing!, two is missing!))
```

As you can see, there are also some conveniences here that cats provides:

- The `EitherNec[A, B]` type alias which is short for `Either[NonEmptyChain[A], B]`.
- The `.toRightNec[A]` method which converts an `Option[B]` to an `EitherNec[A, B]` by providing it with an error value to use if the value is `None`. This also means that we don't need to specify a singleton list when we provide our error value.

There are other convenience methods too:

- `.toLeftNec[B]` which is the equivalent to `toRightNec[A]` above.
- `.rightNec[A]` and `.leftNec[B]` which are available on any type to live them into an `EitherNec`.

All of these convenience methods are available for both `EitherNec` and `EitherNel` with their appropriate suffix.

### Parallel transitivity

It turns out that the conversion between `Parallel` types is transitive. We can use this to our advantage and define our own types to represent programs that take a `Map[String, Int]` as input and return an `EitherNel[String, A]`

```scala
def get(key: String): ReaderT[EitherNec[String, *], Map[String, Int], Int] =
  ReaderT(_.get(key).toRightNec(s"$key is missing!"))

val sequentialProgram: Map[String, Int] => EitherNec[String, Int] =
  (get("one"), get("two")).mapN(_ + _).run

val parallelProgram: Map[String, Int] => EitherNec[String, Int] =
  (get("one"), get("two")).parMapN(_ + _).run

sequentialProgram(Map.empty) // Left(Chain(one is missing!))
parallelProgram(Map.empty) // Left(Chain(one is missing!, two is missing!))
```

> Note: in this section we use the [kind-projector](https://github.com/typelevel/kind-projector) Scala plugin to make the type syntax more bearable.

There's a lot going on here and I don't want to get into `ReaderT` as that is _a whole other topic_. Essentially, because there is a `Parallel` instance for `Either[String, A]` there is also an instance for `ReaderT[EitherNel[String, *], Map[String, Int], A]` which lets us compose instances of this in sequence or in parallel! Neat!