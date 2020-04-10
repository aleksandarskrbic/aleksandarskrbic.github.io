---
title: "Data processing with Akka Actors: Part II"
date: 2020-04-10
header:
  image: "/images/akka/akka-h1.jpg"
tags: [scala, akka, jvm, big data]
excerpt: "Scala, Akka, JVM, Big Data"
---

{: .text-left}

In the [first article]({% post_url 2020-04-04-akka-actors-1 %}) in the *Data processing with Akka Actors series*, I have introduced a problem that we are trying to solve. There was some talk about actors and how to organize them into an actor hierarchy. Then we got deep into details while implementing two simple actors with Scala describing message flow protocol between them and explaining the relationships among actors in the defined actor hierarchy.

*Supervisor Actor* and *Ingestion Actor* are already implemented and discussed in detail and if you remember the actor hierarchy diagram from the [first article]({% post_url 2020-04-04-akka-actors-1 %}) you know that we are going to implement two more actors: *Master Actor* and *Worker Actor*. Since *Master Actor* is the brain of our application, who is responsible for distributing incoming requests to workers and gathering results from them when data processing is finished, most complex actor among all, we will first implement *Worker Actor*.

## Worker Actor

```scala
object Worker {
  final object ResultRequest
  final case class ResultResponse(id: Int, state: Map[String, Long])
  final case class Date(month: String, year: Integer, hour: Integer)
  final case class Log(ip: String, date: Date, url: String, status: String)

  def props(id: Int) = Props(new Worker(id))
}
```

Here is some already familiar pattern where first is implemented actors companion object, where message protocol is defined and also factory method that returns actor wrapped in a Props class which is immutable configuration object used for creating new actors through *actors system* or *actor context*.  If something is not clear in the previous sentence I recommend reading [this page](https://doc.akka.io/docs/akka/current/actors.html) of Akka official documentation.

Now let's talk about the message protocol. There are four defined messages:
* **ResultRequest**  is a message that is received by *Master Actor*, when there is no data left to process, which means, as a name suggest, that worker actor should return aggregated processing results to *Master Actor*.
* **ResultResponse**  is, as you assumed,  a message that contains all data processed by a worker.
* **Date** and **Log** case classes are our domain objects which are consturcted from **Line** message received by *Master Actor*.

By now, as we are already familiar with a message flow through actors, we know that **Line** is just a wrapper class around line read from source by *Ingestion Actor* sent to *Master Actor* and finally forwarded to one of the *Worker Actors*. The goal of our application is to count the number of occurrences of HTTP status codes from file reading it line by line. So upon receiving **Line**,  *Worker Actor* should transform that to  **Log** case class. So here is an implementation of that function:

```scala
trait WorkerHandler {
  import Worker._

  def toLog(line: String): Log = line.split(",").toList match {
    case ip :: time :: url :: status :: _ =>
      val date = time.substring(1, time.length).split("/").toList match {
        case _ :: month :: timeParts :: _ =>
          val year = timeParts.split(":")(0).toInt
          val hour = timeParts.split(":")(1).toInt
          Date(month, year, hour)
      }
      Log(ip, date, url, status)
  }
}
```
Here is a simple function that just parses string that is extracted from **Line** case class and transforms it to **Log** case class. My implementation relies heavily on pattern matching because I like the way it allows us to deconstruct case classes and collections. If you want to found out more about pattern matching in Scala or you are not very clear about it, feel free to reach me or read [this](https://docs.scala-lang.org/tour/pattern-matching.html).

Finally, let's review *Worker Actor* implementation:

```scala
class Worker(id: Int) extends Actor
 with ActorLogging
 with WorkerHandler {
  import Worker._

  type StatusCode = String
  type Count = Long

  var state: Map[StatusCode, Count] = Map.empty

  override def receive: Receive = {
    case Ingestion.Line(text) =>
      val status = toLog(text).status
      state.get(status) match {
        case Some(count) =>
          state += (status -> (count + 1))
        case None =>
          state += (status -> 1)
      }
    case ResultRequest =>
      sender() ! ResultResponse(id, state)
  }
}
```

There are only a few lines of code, but to be sure let's examine all details.

Starting from the beginning we could see **state** variable where processing results are stored.

```scala
var state: Map[StatusCode, Count] = Map.empty
```

Upon receiving **Line** message, *Worker Actor* transform it to **Log** and then update state with next logic:

```scala
state.get(status) match {
  case Some(count) =>
    state += (status -> (count + 1))
  case None =>
    state += (status -> 1)
}
```

Here we have some pattern matching again.
Since *Map* from Scala Collections returns *Option[V]*, we are able to pattern match against it. If there is some value we will get *Some(value)* and if *Option* is empty we will get *None*. To learn more about *Option type* check [this](https://www.scala-lang.org/api/current/scala/Option.html).

And that is a full implementation of a *Worker Actors*, simple as that.

## Master Actor

This is the last chapter of this article where *Master Actor* implementation is going to be reviewed.

```scala
object Master {
  final object Initialize
  final object MasterInitialized
  final object CollectResults
  final case class Aggregate(result: Seq[(String, Long)])

  def props(nWorkers: Int) = Props(new Master(nWorkers))
}
```

Since *Master Actor* have multiple behaviors, each will be explained separately. Let's start with the initial behavior.

```scala
override def receive: Receive = {
  case Initialize =>
    log.info(s"Spawning $nWorkers workers...")
    val workers: Seq[ActorRef] = (1 to nWorkers).map(createWorker)
    context.become(forwardTask(workers, 0))
    sender() ! MasterInitialized
}
```

*Initialize* message that is received from *Ingestion Actor* will trigger the creation of *Worker Actors*. Number of workers is defined upon creating *Master Actor*. Here is the implementation of *createWorker* method:

```scala
def createWorker(index: Int): ActorRef = 
  context.actorOf(Worker.props(index), s"worker-$index")
```

After workers are initialized, they are ready to start receiving tasks from their parent.

```scala
context.become(forwardTask(workers, 0))
```

This code snippet(above) basically tells actors how to react to the next message.
Actors *context.become* method takes a *PartialFunction[Any, Unit]* that implements the new message handler.

```scala
type Receive = PartialFunction[Any, Unit]
```
*Receive* type that is seen in code is just Akka alias for *PartialFunction[Any, Unit]*.

Now, after this is clear, let's review *forwadTask* behavior:

```scala
def forwardTask(workers: Seq[ActorRef],
                currentWorker: Int): Receive = {
  case line @ Ingestion.Line(_) =>
    val worker = workers(currentWorker)
    worker ! line
    val nextWorker = (currentWorker + 1) % workers.length
    context.become(forwardTask(workers, nextWorker))
  case CollectResults =>
    workers.foreach(_ ! Worker.ResultRequest)
    context.become(collectResults())
}
```
In this behavior *Master Actor* react on two type of messages: *Line* and *CollectResults*.

When *Line* is received, we will have access to *workers* which is collection of all *Worker Actors* and *currentWorker*  which is the index of *Worker Actor* to whom the incoming message will be passed. After the message is sent to the wanted worker, we need to determine the index of the next *Worker Actor* who will need to receive the next message.

```scala
val nextWorker = (currentWorker + 1) % workers.length
context.become(forwardTask(workers, nextWorker))
```

In the first line of code *nextWorker* index is determined in a round-robin fashion, and the next line is the same as one we explained before. You can capture the pattern that we are using to carry the state via method arguments.

When *CollectResults* message is received, we just iterate over *workers* and request results from them, after that *Master Actor* behavior is switched to *collectResults*, when it's ready to handle responses from all *workers*.

```scala
val results = new ArrayBuffer[Worker.ResultResponse]()

def collectResults(): Receive = {
  case response @ Worker.ResultResponse(_, _) =>
    results += response
    if (results.length == nWorkers) {
      context.parent ! toAggregate(results.toSeq)
    }
}
```

Here you can see that we introduced *results* which is a collection of all received messages from workers. When all results are collected from workers, we will transform responses to the final result and pass it to the parent(*Ingestion Actor*).

```scala
trait MasterHandler {
  def toAggregate(results: Seq[Worker.ResultResponse]): Aggregate = {
    val aggregate = results
      .map(_.state)
      .flatMap(_.toList)
      .groupBy(_._1)
      .map { case (k, v) => k -> v.map(_._2).sum }
      .toList
      .sortBy(_._2)
    Aggregate(aggregate)
  }
}
```

In *MasterHandler* there is a method that transforms results to *Aggregate* case class, which is just a container for a collection of *Tuple(String, Long)*, where the first element of a tuple is a *status code*, and the second one is a *count of occurrences*.

And finally, we finished our application. Let's review the whole code for *Master Actor*:

```scala
class Master(nWorkers: Int) extends Actor
 with ActorLogging
 with MasterHandler {
  import Master._

  override def receive: Receive = {
    case Initialize =>
      log.info(s"Spawning $nWorkers workers...")
      val workers: Seq[ActorRef] = (1 to nWorkers).map(createWorker)
      context.become(forwardTask(workers, 0))
      sender() ! MasterInitialized
  }

  def forwardTask(workers: Seq[ActorRef],
                  currentWorker: Int): Receive = {
    case line @ Ingestion.Line(_) =>
      val worker = workers(currentWorker)
      worker ! line
      val nextWorker = (currentWorker + 1) % workers.length
      context.become(forwardTask(workers, nextWorker))
    case CollectResults =>
      workers.foreach(_ ! Worker.ResultRequest)
      context.become(collectResults())
  }

  val results = new ArrayBuffer[Worker.ResultResponse]()

  def collectResults(): Receive = {
    case response @ Worker.ResultResponse(_, _) =>
      results += response
      if (results.length == nWorkers) {
        context.parent ! toAggregate(results.toSeq)
      }
  }

  def createWorker(index: Int): ActorRef =
    context.actorOf(Worker.props(index), s"worker-$index")
}
```

## Summary

This is the last part of *Data processing with Akka Actors* series. Here is the source code if you want to play with it or even improve it.
Github repository: [Data processing with Akka Actors](https://github.com/aleksandarskrbic/akka-actors-blog).

You learned how to write a relatively simple application with Akka, how to design master-worker architecture and how to implement a few communication patterns between actors. There is a lot of improvements that can be added to this application. Some of them are to refactor codebase to use Typed Actors and to implement supervision strategy for actors. Check official Akka documentation on [Supervision and Monitoring](https://doc.akka.io/docs/akka/2.6/general/supervision.html) to learn more about it. Another way to improve this application will be to completely remove *Master Actor* and to use [Akka Routers](https://doc.akka.io/docs/akka/current/routing.html), that can easily help us to distribute request among workers without worrying about algorithms like round-robin or some else, without need to implement them on your own, as we did here for the purpose of learning.

Some of these improvements will be a theme for another article.

You can find me at:
* [Linkedin](https://www.linkedin.com/in/aleksandar-skrbic/)
* [Github](https://github.com/aleksandarskrbic)

Or just send me a question to [skrbic.alexa@gmail.com]()
