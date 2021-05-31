Akka is a suite of libraries for Scala and Java which provides tools for organising asynchronous programs.

Here I'll be focusing on using akka with Scala.

## What is akka streams?

Akka streams is a part of the Akka ecosystem which implements the concept of [reactive streaming](https://www.reactive-streams.org/). It is fully compatible with other implementations of the reactive streams specification.

## Why akka streams?

There are lots of good reasons to use akka streams. I'll go through a few of them here but this is by no means an exhaustive list:

- Modelling computations as streams of data elements being transformed can be extremely intuitive, especially for people who like functional / declarative programming.
- Back pressure helps build systems which can respond to load in a predictable way.
- Akka is compatible with other reactive streams implementations which allows for interoperability of graphs and stages.
- Even small systems built on the actor model can be difficult to reason about as a whole. By creating a graph to represent the same computation this can be much easier to reason about, test and maintain.

## Graphs

At the core of akka streams is the concept of a graph. Like in the mathematical sense, a graph is a set of **nodes** connected by **edges**. In our case, each node is a processing step in our computation, and an edge indicates data flowing from one node to another.

Each `Graph` in akka is a blueprint, or a piece of a blueprint of a program that we can run.

Let's dig in and take a look at `Graph` in more detail.

```scala
type Graph[+S <: Shape, +M]
```

There are two type parameters here that are going to take some breaking down.

### `S` Shape

The `Shape` type parameter indicates the external interface of the `Graph`.

To get a feel for what this means let's take a look at some of its implementations.

```scala
object ClosedShape extends Shape
final case class SourceShape[+T](out: Outlet[T]) extends Shape
final case class SinkShape[-T](in: Inlet[T]) extends Shape
final case class FlowShape[-I, +O](in: Inlet[I], out: Outlet[O]) extends Shape
```

> Note: These aren't _all_ of the implementations included with akka streams but they're illustrative. In fact, you can even create your own `Shape` to suit your needs.

Essentially, a `Shape` has a number of inlets and outlets which determine how it can be combined with other `Graph`s. Let's examine each of these implementations in turn:

- `ClosedShape` this shape has no inlets or outlets and therefore represents a stand-alone graph which cannot be composed with other graphs.
- `SourceShape[T]` has an outlet but no inlet. This shape of graph is a _source_ of data (hence the name) and can be composed with any graph which has a free inlet.
- `SinkShape[T]` has an inlet but no outlet. This shows us that data can flow _in to_ this graph but not out of it.
- `FlowShape[I, O]` has both an inlet and an outlet. Its inlet and outlet have different types which shows us that this graph can output a different type of element than it consumes. For example, a function could be applied to the input to produce each output element, much like `.map` on standard Scala collections.

### `M` Materialized Value

The `M` type parameter is known as the materialized value of the graph. This represents the type of value that is produced when the stream is run (a.k.a. _materialized_).

Materialized values can be anything but there are common types that crop up often:

- `NotUsed` this is the materialized value of graphs which do not produce any meaningful result when materialized. This is essentially the same as `Unit` from a Scala perspective. However, as akka is cross compatible with Scala and Java it is preferable to use `NotUsed` as `Unit` is a Scala specific type. In my experience this is the most common materialized value you'll find.
- `Future[A]` as materialized values are immediately produced when a stream is run, many graphs produce `Future` values that complete once the stream itself finishes processing.
- `Future[Done]` this is a common example of the above where a future is used to indicate the stream has finished processing but does not return anything meaningful as a result. This is useful as it allows the caller to sequence operations after the completion of the stream.

## Running streams

In order to run a graph we need an instance of a `Materializer`. There is an implicit conversion from akka's `ActorSystem` to `Materializer`.

> Note: If you use akka streams in a Play framework project, there is a `Materializer` created from the default `ActorSystem` available by default through dependency injection.

Rather than representing a computation itself, each `Graph` is a blueprint of a computation which can be materialized any number of times.

In the following sections we'll need to materialize our own graphs so I have included the below boilerplate which can be used to run examples and to experiment. For brevity I will omit the boilerplate from the examples themselves.

```scala
import akka.NotUsed
import akka.actor.ActorSystem

// imports for common akka streams methods
import akka.stream._
import akka.stream.scaladsl._

// imports for evaluating `Future`s
import scala.concurrent.Future
import scala.concurrent.Await
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

object AkkaApp extends App {

  implicit val system: ActorSystem = ActorSystem()
 
  try {

    // our code here

  } finally system.terminate()
}
```

## Closed graphs

In order to materialize a graph, it must be closed (i.e. have no open inlets or outlets).

```scala
val materializer: Materializer = implicitly
val graph: Graph[ClosedShape, Future[Int]] = Source.single(1).toMat(Sink.head)(Keep.right)
val future: Future[Int] = materializer.materialize(graph)
val result: Int = Await.result(future, 1.second)
println(result)
```

Don't worry about the graph creation syntax, we'll get to that in a moment.

First, let's step through this:
1) Fetch our materializer from implicit scope.
2) Create a `Graph[ClosedShape, Future[Int]]`.
3) Materialize the graph into a `Future[Int]`.
4) Await the result of the `Future[Int]`.
5) Print the result to the console.

This should print `1` to the console.

This is a perfectly acceptable way to materialize a graph, however, we can tidy up this code a little:

```scala
val graph: RunnableGraph[Future[Int]] = Source.single(1).toMat(Sink.head)(Keep.right)
val future: Future[Int] = graph.run()
val result: Int = Await.result(future, 1.second)
println(result)
```

Most of the methods that return a closed graph will specifically return a `RunnableGraph[M]` which gives us access to the `.run()` method.

```scala
final case class RunnableGraph[+M](...) extends Graph[ClosedShape, M] {
  def run()(implicit materializer: Materializer): M = materializer.materialize(this)
}
```

## Composing linear graphs

Let's get started building our own graphs!

In akka streams, individual graph nodes are often called operators and they come in 3 kinds:

- `Source[Out]`
- `Flow[In, Out]`
- `Sink[In]`

These match the `Shape`s that we talked about earlier. I've labelled the type parameters here to match the flow of data, for example: Data flows _out_ of a source and _in_ to a sink. None of these graphs are _closed_ so they can't be materialized by themselves.

In order to make runnable graph, we can compose different operators.

I've omitted the materialized values in the examples below to simplify them. We'll cover what happens to the type of materialized values once we know how to combine graphs.

### Source ~> Sink

The simplest runnable graphs we can make are by connecting a `Source` directly to a `Sink`. We can do this with the `.to` method.

```scala
val source: Source[Int, _] = Source(List(1, 2, 3))
val sink: Sink[Int, _] = Sink.fold[Int](0)(_ + _)
val graph: RunnableGraph[_] = source.to(sink)
```

When combining operators like this, the types of the inlets/outlets must match up. In this case the outlet of the `Source` and the inlet of the `Sink` both have the type `Int`.

### Source ~> Flow

Not all graphs compose to make `RunnableGraph`s, composing a `Source` with a `Flow` gives us a `Source`. To connect a `Source` and a `Flow` we use the `.via` method.

```scala
val source: Source[Int, _] = Source(List(1, 2, 3))
val flow: Flow[Int, String, _] = Flow[Int].map(_.toString)
val graph: Source[String, _] = source.via(flow)
```

Here we combine a `Source` which produces `Int`s with a `Flow` that turns each `Int` into a `String`. This returns a `Source` that produces `String`s. This makes sense because although we've paired up the `Source`'s outlet with the `Flow`'s inlet, the `Flow`'s outlet hasn't been attached to anything. That leaves us with a graph with a single outlet of `String`, a graph with a single outlet is a `Source`!

### Flow ~> Sink

As with the above we can create a `Sink` by combining a `Flow` and a `Sink`. Take note of the types of the elements flowing through each operator. Because we are attaching something to a `Sink` we use `.to`.

```scala
val flow: Flow[Int, String, _] = Flow.fromFunction(_.toString)
val sink: Sink[String, _] = Sink.ignore
val graph: Sink[Int, _] = flow.to(sink)
```

### Flow ~> Flow

Finally, we can chain together `Flow`s to create another flow, this works in the same way that function composition does:

```scala
val f: Int => String = _.toString
val g: String => Char = _.head
val h: Int => Char = f.andThen(g)

val ff: Flow[Int, String, _] = Flow.fromFunction(_.toString)
val gg: Flow[String, Char, _] = Flow.fromFunction(_.head)
val hh: Flow[Int, Char, _] = ff.via(gg)
```

These methods compose in data flow order, from source to flow to `Sink`. 

`.via` is used to connect most operators, but when you connect an operator to a `Sink` you use `.to`.

For example:

```scala
source
	.via(flow1)
	.via(flow2.via(flow3))
	.to(sink)
```

### Selecting materialized values

Up to now, when combining graphs we haven't considered what happens to the materialized value. Let's look at `.to` and `.via` in more detail to see how they work:

```scala
final class Source[+Out, +M] extends Graph[SourceShape[Out], M] {
	def to[M2](sink: Graph[SinkShape[Out], M2]): RunnableGraph[M]
	
	def via[T, M2](flow: Graph[FlowShape[Out, T], M2]): Source[T, M]
}
```

> Note: This is simplified slightly from the code in akka streams itself.

So, when you compose graphs with these methods, the resulting graphs always have a materialized value of the graph from the left hand side.

This means that if you composed a bunch of graphs together in data flow order you'd end up with the materialized value of the initial source. But what if you want to keep the materialized value on the right? Or what if you want to keep both?

```scala
final class Source[+Out, +M] extends Graph[SourceShape[Out], M] {
	def toMat[M2, M3](sink: Graph[SinkShape[Out], M2])(combine: (M, M2) => M3): RunnableGraph[M]
	
	def viaMat[T, M2, M3](flow: Graph[FlowShape[Out, T], M2])(combine: (M, M2) => M3): Source[T, M]
}
```

All the methods for composing operators have two forms:
- `x` which retains the materialized value on the left and
- `xMat` which accepts an extra parameter `combine: (M, M2) => M3` that is used to determine what the materialized value should be for the new graph.

The combine function can be anything, but there are a few cases that are extremely common:
- Pick the value from the left
- Pick the value from the right
- Tuple both together
- Return `NotUsed`

These cases are _so_ common that akka has optimised function instances that do this! `Keep.left`, `Keep.right`, `Keep.both`, and `Keep.none` respectively. As with `NotUsed` these have the added convenience of conveying intent and being available in both Java and Scala.

Although these are the most common ways of composing operators, it's good to remember that you can use arbitrary functions to produce the new materialized value. For example, imagine we have two graphs which both materialize into `Future`s. we can use a function to combine them in the graph definition itself:

```scala
def combine[A, B](fl: Future[A], fr: Future[B]): Future[(A, B)] =
  for {
    l <- fl
    r <- fr
  } yield (l, r)

val source: Source[Int, NotUsed] =
  Source(List(1, 2, 3))

val countSink: Sink[Int, Future[Int]] =
  Sink.fold(0)((m, _) => m + 1)

val collectSink: Sink[Int, Future[List[Int]]] =
  Sink.collection[Int, List[Int]]

val graph2: RunnableGraph[Future[(Int, List[Int])]] =
  source
    .alsoToMat(countSink)(Keep.right)
    .toMat(collectSink)(combine)

println(Await.result(graph2.run(), 1.second))
```

This graph outputs a single future of a tuple containing the count of all the elements that have passed through it and a `List` of the collected elements at the end.

## Demand and back pressure

Now that we have a good understanding of graphs, we should talk about back pressure - the _reactive_ part of reactive streams.

Let's look at a few examples

### Slower `Source`

Imagine we have a `Source` which produces data slowly. For example, a weather report updated twice a day, and a `Sink` which prints that report to the console.

Even without reactive streaming, most systems would deal with this scenario. For example, if we represented our `Source` and `Sink` as actors, the sink actor can _always_ handle the amount of data that the source is sending to it - everything is fine!

### Faster `Source`

Imagine we have a `Source` which will produce data super quickly. For example, a stream of tweets from a particularly vocal user, and a `Sink` which would send each tweet to a physical printer one at a time.

In this case we would start to see problems if we tried to model this with actors. As the source is producing data faster than the sink can consume it, the sink's mailbox will start to fill up and eventually overflow. Depending on the configuration of the system this could result in a loss of data, or the entire system could fail - That's pretty bad!

### Demand

In akka streams, we handle these cases explicitly with the concept of _demand_:

- A `Source` will only _emit_ elements when it knows there is demand from downstream.
- A `Sink` will only  _request_ elements when it knows it can handle them.

Demand is also referred to as back pressure. Think of it like this, a `Sink` will push back against a source until it's ready to consume data and a `Source` will not emit while it is pushed against.

> Note: We say back pressure as this mechanism is similar to pressure in fluid dynamics.

How demand is implemented depends on the specifics of the `Source` or `Sink` in question. Let's look at how this would apply to our above examples.

### Demand for `Source`s

Our super-fast tweet source would need to make sure it _only emits when there is demand_. To do this we need to decide how we would turn a stream of lots of tweets into a stream of fewer tweets.

One way to do this would be to only poll for tweets when we receive demand. However, to prevent us emitting duplicate elements we'd need to maintain a buffer of the last element to compare in care there have been no updates since since demand was last signalled.

Another way would be to use a buffer to store tweets waiting to be emitted once demand is signalled. However, what happens when that buffer fills up? We only have finite memory to deal with so just adding elements to a buffer isn't really enough. We'll cover how to deal with that shortcoming in a moment.

### Demand for `Sink`s

Our sloth-like printer sink would need to be implemented to make sure that we _only_ signal demand once we're sure we can handle the next element in the stream.

The specifics of this are only _really_ relevant when implementing a new graph stage (which I won't talk about in this article) however, it's good to think about this conceptually.

In blocking, imperative pseudocode we could think of an implementation something like this:

```
setup()
while(!upstreamCompleted) {
	demand()
	getNextElement()
	print()
}
shutdown()
```

This kind of implementation is generally preferable to using buffers as it doesn't have a failure case to consider, it always back pressures until it can handle a new element

### Buffers

When defining akka streams graphs we can add explicit buffers to specify how demand should be handled for certain parts of our graphs, for example:

```scala
val source: Source[_, _] = ???
source.buffer(10, OverflowStrategy.fail)
```

As you can see, the `buffer` method takes 2 parameters:

- The size of the buffer.
- The strategy that should be employed when the buffer is exceeded.

In my experience, this explicit boundedness and error handling forces us to think about these aspects of a system at the design stage which leads to more predictable and maintainable systems.

The different values for `OverflowStrategy` are:

- Discard all new elements (`dropNew`)
- Replace the newest element of the buffer (`dropTail`)
- Discard the oldest element of the buffer (`dropHead`)
- Cause this part of the graph to fail (`fail`)
- Drop the whole buffer and start buffering again from scratch (`dropBuffer`)
- Backpressure the rest of the upstream graph (`backpressure`)

Earlier, I mentioned that only closed graphs can be materialized. With our new understanding of demand hopefully that makes more sense:

A graph with an open inlet would have nowhere to pull data from, and a graph with an open outlet would never receive demand. In either case no data can flow, so it makes sense that akka streams' API only lets you materialize closed graphs.

## Conclusion

So far we've only covered the basics of working with akka streams but hopefully this has been useful for anyone trying to understand the basic concepts of akka streams and reactive streaming.

In the future I'd like to cover some more advanced topics such as:
- Creating non-linear and potentially cyclic graphs.
- Creating custom graph stages.
- Unit testing graphs.
- Handling failure in running graphs.
- Real world examples.
- Exercises.

But this article has already gotten too long! Feel free to leave any comments or corrections in the [issues section on this repo](https://github.com/wolfendale/wolfendale.github.io/issues)