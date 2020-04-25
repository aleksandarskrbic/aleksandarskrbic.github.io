---
title: "Data processing with Akka Actors: Part I"
date: 2020-04-04
header:
  image: "/images/akka/akka-h1.jpg"
tags: [scala, akka, jvm, big data]
excerpt: "Scala, Akka, JVM, Big Data"
mathjax: "true"
---

{: .text-left}

Everyone experienced in a field of distributed and data-intensive systems or big data technologies,
in general, must have heard about [Akka](https://akka.io/).
This article will be part of the series, where I will try to introduce you to the actor programming model while developing a data processing application exploiting techniques provided by *Akka Actors library* to build concurrent and parallel systems. This series will skip *"Scala and Akka Basics"* parts because official documentation is great and you can build a good foundation of Akka by reading it.  The goal of this series is to provide a hands-on introduction to the Akka toolkit.

So, let’s talk about the implementation details. As you guessed Scala is going to be used with Akka instead of Java and I think there is no need for the explanation.
An application that is going to be developed will be reading logs from the file system and count the number of occurrences of HTTP status codes in a given file. All data processing tasks will be executed in parallel.
In the actor programming model, the base unit is an actor obviously, and actors communicate with each other in an asynchronous manner, by sending and receiving messages.

Since our application will be actor based let’s take a look at the actor hierarchy.

<img src="{{ site.url }}{{ site.baseurl }}/images/akka/actor-hierarchy.jpg">

You can see that there are four actor types, *Supervisor, Ingestion, Master, and Worker*. Names are more or less self-explanatory, but let's make things clear. Good practice in actor-based programming is to organize actors in a tree-like structure. So the root actor in our application will be
*Supervisor Actor* who will be an entry point in our system. 

* *Supervisor Actor* is responsible for spawning and managing *Ingestion Actor*. 
* *Ingestion Actor*  will be the parent of *Master Actor*. 
* Also, you can notice that we are going to have multiple *Worker Actors*, who are going to do all the heavy lifting, but *Master Actor* is responsible for their coordination.

To give you more details about the problem that we are going to solve using Akka let me walk you through the whole data processing flow.
* Dataset of weblogs that we are going to process can be downloaded from [kaggle](https://www.kaggle.com/shawon10/web-log-dataset).
* It's csv file that contains *IP, Time, URL, Status* on every line, but not every line is in a valid form, so we will have to deal with that.
* When *Ingestion Actor* is initialized, it will try to initialize *Master Actor*, who will spawn an arbitrary number of *Worker Actors*. After workers are initialized, *Master Actor* will notify *Ingestion Actor* that it's ready to start processing.
* *Ingestion Actor* will start reading from a given file line by line, filter only valid ones, and pass it to *Master Actor*.
* *Master Actor* will distribute incoming requests from *Ingestion Actor* to *Worker* actors in a round-robin fashion. 

Finally, let's review some code.

## Applcation entry point

```scala
object Application extends App {

  implicit val system = ActorSystem("actor-system")

  val supervisor = system.actorOf(
    Supervisor.props("input", "output", 3), "supervisor"
  )
  supervisor ! Supervisor.Start
}
```

## Supervisor Actor

```scala
object Supervisor {
  final object Start
  final object Stop

  def props(input: String, output: String, parallelism: Int) =
    Props(new Supervisor(input, output, parallelism))
}
```

In a *Supervisor Actor companion object*, all messages and methods for creating actor are defined. This pattern should be applied to every actor.

Here is *Supervisor Actor* implementation:

```scala
class Supervisor(input: String,output: String, parallelism: Int)
  extends Actor
  with ActorLogging {
  import Supervisor._

  val ingestion: ActorRef = createIngestionActor()

  override def receive: Receive = {
    case Start =>
      ingestion ! Ingestion.StartIngestion
    case aggregate @ Master.Aggregate(_) =>
      aggregate.result.foreach(println)
    case Stop =>
      context.system.terminate()
  }
}
```

The Supervisor Actor is simple. He is responsible for starting the whole data processing pipeline,  printing results to console upon receiving it, and stopping the actor system. For the sake of simplicity, I choose just to print the results, but in a real scenario, it could be something like writing results to some database, message queue, or filesystem. Also shutting down the actor system upon finishing all tasks is very important since it a heavyweight structure which upon initialization allocates threads, and to release them you need to stop the actor system.

## Ingestion Actor

*Ingestion Actor* is responsible for reading a file from a given path, process it, and pass it to a *Master Actor*. When the whole file is read, it will notify *Master Actor* that ingestion is done.

*Ingestion Actor companion object* implementation:

```scala
object Ingestion {
  final object StartIngestion
  final object StopIngestion
  final case class Line(text: String)

  def props(input: String, output: String, nWorkers: Int) =
    Props(new Ingestion(input, output, nWorkers))
}
```

For those actors who have some additional functionalities, I prefer to implement an additional trait, where business logic is concentrated and will be just mixed with the actor. This makes the actor class clean and simple because in it we are only dealing with behavior. Actor behavior is a way of message processing.

```scala
trait IngestionHandler {
  val ip: Regex = """.*?(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3}).*""".r
  val validIp: String => Boolean = line => ip.matches(line.split(",")(0))
}
```
Here we have a simple function to validate if the line starts with an IP address or not, since we need only those lines that do.

And finally, *Ingestion Actor* implementation:

```scala
class Ingestion(input: String, output: String, nWorkers: Int)
  extends Actor 
  with ActorLogging
  with IngestionHandler {
  import Ingestion._

  val master: ActorRef = createMasterActor()
  lazy val source = Source.fromFile(createFile())

  override def receive: Receive = {
    case StartIngestion =>
      log.info("Initializing Master Actor...")
      master ! Master.Initialize
    case Master.MasterInitialized =>
      log.info("Starting ingestion...")
      source.getLines().filter(validIp).map(Line).foreach(master ! _)
      log.info("Collecting results...")
      master ! Master.CollectResults
    case aggregate @ Master.Aggregate(_) =>
      context.parent.forward(aggregate)
      self ! StopIngestion
    case StopIngestion =>
      source.close()
      context.parent ! Supervisor.Stop
  }
```

Upon initialization, *Master Actor* is spawned, and the data source is ready. When **MasterInitialized** message is received, *Ingestion Actor* start to read file line by line, filter only the valid ones, map them into **Line** case class and pass it to *Master Actor*. After the whole file is read,
*Ingestion Actor* demands results from  *Master Actor*. Note that all communication is fully asynchronous. Received results will be forwarded to the parent actor (*Supervisor Actor*). After that, the file stream is closed and the parent is notified about that with message **Stop**.

## Summary

In this article, I introduced the problem that we are trying to solve and the technology stack that is going to be used. Also, I hope you gained some basic understanding of how to implement actors and design an Actor System.

This is it for now. In the next part of the series, we are going to review first *Worker Actor* and finally *Master Actor*, which is probably the most complicated part of the system.

You can find me at:
* [Linkedin](https://www.linkedin.com/in/aleksandar-skrbic/)
* [Github](https://github.com/aleksandarskrbic)

Or just send me a question to [skrbic.alexa@gmail.com]()