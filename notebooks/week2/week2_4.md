# Designing Actor Systems

## Starting Out

- Imagine giving the task to a group of people, dividing it up.
- Consider the group to be of very large size.
- Start with how people with different tasks will talk with each other.
- Consider these “people” to be easily replaceable.
- Draw a diagram with how the task will be split up, including communication lines

## Example: the Link Checker

Write an actor system which given a URL will recursively download the
content, extract links and follow them, bounded by a maximum depth; all links encountered shall be returned

### Plan of Action
* Write web client which turns a URL into a HTTP body asynchronously. We will be using ”com.ning” % ”async-http-client” % ”1.7.19”
* Write a Getter actor for processing the body.
* Write a Controller which spawns Getters for all links encountered.
* Write a Receptionist managing one Controller per request.


There will be one actor which we call receptionist. This one is responsible for accepting incoming requests. Request comes from client. Receptionist is responsible for noting down the client, request and telling someone else to do the job. In web, links can be cycles. When we see a link we already visited, we need to stop or we run into endless loop. One such person will remember the links we visited. Let's call such actor controller. Controller remembers what's visited and still needs visiting. It would be better to have someone else to have the job of visiting. Let's call that actor Getter. Getter visits a URL retrieve the documents, extract the links which are in the document and tell controller what it has found. The controller, then spawn other getters to visit the new links and so on. 

To recap, let's put the messages which will be used to achieve this.  The client sends `get(url)` request for a URL, the Receptionist will create a controller and send it a `check(url, depth)`. The controller then tell Getter to retrive what is at the URL `Get(url)`, and the Getter then reply with possibly multiple links of type `link` and finally `done`. All links found in URL should be treated quickly and can be visited parallel. So there will be multiple Getters. The controller needs to keep track which URL was encountered at which depth. Once the depth is exhausted the final result is communicated to the receptionist, keep track which client send URL and send the answer. 

Let's start simple.
```scala
val client = new AsyncHttpClient
def get(url: String): String = {
val response = client.prepareGet(url).execute().get
if (response.getStatusCode < 400)
response.getResponseBodyExcerpt(131072)
else throw BadStatus(response.getStatusCode)
}
```
The problem is in line, `val client = ...`. `execute()` returns a Future and calling get method returns the string synchronously. But it
blocks the calling actor until the web server has replied:
* actor is deaf to other requests, e.g. cancellation does not work
* wastes one thread—a finite resource


Let's fix this.

```scala

private val client = new AsyncHttpClient
def get(url: String)(implicit exec: Executor): Future[String] = {
val f = client.prepareGet(url).execute();
val p = Promise[String]()
f.addListener(new Runnable {
def run = {
val response = f.get
if (response.getStatusCode < 400)
p.success(response.getResponseBodyExcerpt(131072))
else p.failure(BadStatus(response.getStatusCode))
}
}, exec)
p.future
}
```
First we stop at `execute` on `f`. This gives us back a future. We want to adapt this into Scala future so we construct a promise of String. The future returned by `execute` is not a `java.util.concurrent` Future, it has some added functionality, namely you can added a listener. When the future is completed, a runnable is registered on the listener, which will run. We require executor to run it. We get future from Promise `p.future`. `AsyncHttpClient` is a Java library using Java and it's own futures. Basically we mapped from listenable `AsnycHttpClient` future to Scala futures.

If you have event based source for something and you want to wait single shot event in this case, it is best to wrap it in future and expose it as API.  



> A reactive application is non-blocking & event-driven top to bottom.

Now we now how to retrieve documents from web, we need to find links int HTML. For that we use Java library called JSoup. Parsing a body string, returns a structured representation of HTML `document`. We can query `document` with all anchor tags, we then grab a iterators and convert to Scala iterator. We then use further iterators we return absolute URL in href attributes. This gives URL for further sites to visit.

```scala
import org.jsoup.Jsoup
import import scala.collection.JavaConverters._
def findLinks(body: String): Iterator[String] = {
val document = Jsoup.parse(body, url)
val links = document.select(”a[href]”)
for {
link <- links.iterator().asScala
} yield link.absUrl("href")
}
```
```scala

class Getter(url: String, depth: Int) extends Actor {
implicit val exec = context.dispatcher
WebClient get url pipeTo self
...
}
```
`context.dispatcher` the machinery that runs the actor itself,  of `Getter` can be used to run both Java and Scala futures.

```scala
class Getter(url: String, depth: Int) extends Actor {
implicit val exec = context.dispatcher
WebClient get url pipeTo self
def receive = {
case body: String =>
for (link <- findLinks(body))
context.parent ! Controller.Check(link, depth)
stop()
case _: Status.Failure => stop()
}
def stop(): Unit = {
context.parent ! Done
context.stop(self)
}
}
```
`context` has a field `parent`. Remember every actor has exactly one parent which created it. If we get a string `body` we use `findLinks` to get iterator, and for each link we send them as message to parent actor. Once we communicated all the links to parent, we stop, which means sending parent done message and stopping itself. In case of failure, we stop. 

> Actors are run by a dispatcher—potentially shared—which can also run Futures.

## Actor-Based Logging

* Logging includes IO which can block indefinitely
* Akka’s logging passes that task to dedicated actors
* supports ActorSystem-wide levels of `debug`, `info`, `warning`, `error`
* set level using setting `akka.loglevel=DEBUG `(for example)

```scala
class A extends Actor with ActorLogging {
def receive = {
case msg => log.debug("received message: {}", msg)}
}

```
Logging is also handled by Akka. The solution we chose such that the entity which wants to do logging is not blocking. Pass that off to a dedicated actor. Sending to an actor is non-blocking operation.
The source information provided by logger contains the actor name. That's why it's important to name actors properly. Here we simply log a debug statement.

## The Controller
```scala
class Controller extends Actor with ActorLogging {
var cache = Set.empty[String]
var children = Set.empty[ActorRef]
def receive = {
case Check(url, depth) =>
log.debug(”{} checking {}”, depth, url)
if (!cache(url) && depth > 0)
children += context.actorOf(Props(new Getter(url, depth - 1)))
cache += url
case Getter.Done =>
children -= sender
if (children.isEmpty) context.parent ! Result(cache)
}
}
```
The job of controller is to accept `check(url,depth)` messages for certain URL and once everything is done, send the URL results. The results needs to be collected somewhere so we define `cache` which is set of strings. Here strings are links where it was visited. Whenver `check(url,depth)` arrives we log it at debug level. If cache already contains the url, we don't need to anything. If maximum depth is 0 then we don't need to do anything. But otherwise we need to create new `Getter` with new url, and decrease depth, and add it to children set of `ActorRef`. Now add url to cache since we visited. `Getter` we have just created go to web client get back the links and send other check requests and depth -1. Once it's done we tell `context.parent` the result which is cache.

> Prefer immutable data structures, since they can be shared.

## Handling Timeouts
`Controller` and `Getter` play well together as long as web client works well. If the webserver takes long time to respond, we need to forsee a timeout.
For this, we use another function of  actor context, `setReceiveTimeOut`. This time out is a timer which is reset after processing of each message. So wether we get a check or `Getter.Done`, the receive time out will again reset to 10 seconds. When it expires `RecevieTimeOut` we tell all our children to abort.
```scala
import scala.concurrent.duration._
class Controller extends Actor with ActorLogging {
context.setReceiveTimeout(10.seconds)
...
def receive = {
case Check(...) => ...
case Getter.Done => ...
case ReceiveTimeout => children foreach (_ ! Getter.Abort)
}
}

class Getter(url: String, depth: Int) extends Actor {
...
def receive = {
case body: String =>
for (link <- findLinks(body)) ...
stop()
case _: Status.Failure => stop()
case Abort => stop()
}
def stop(): Unit = {
context.parent ! Done
context.stop(self)
}
}
```


## The scheduler

Akka includes a timer service optimized for high volume, short durations and frequent cancellation.
```scala
trait Scheduler {
def scheduleOnce(delay: FiniteDuration, target: ActorRef, msg: Any)
(implicit ec: ExecutionContext): Cancellable
def scheduleOnce(delay: FiniteDuration)(block: => Unit)
(implicit ec: ExecutionContext): Cancellable
def scheduleOnce(delay: FiniteDuration, run: Runnable)
(implicit ec: ExecutionContext): Cancellable
... // the same for repeating timers
}
```
The focus of such scheduler is support high frequency scheduler tasks but very frequent cancellation of this. But it's not terribly precise. It's main use is to schedule the sending of a message to actor in future point in time, which is first variant above. The object returned is `Cancellable` which you can use to cancel the task. There might be race you firing the task and cancelling.

The other two is for Scala and Java runnig a block of code after delay.

If you want a timeout after the controller starts and not 10 seconds after the message is processed. The context gives you access also to whole system. The system is container in which all actors run. It contains `scheduler` to run this particular code after 10 seconds .

```scala
class Controller extends Actor with ActorLogging {
import context.dispatcher
var children = Set.empty[ActorRef]
context.system.scheduler.scheduleOnce(10.seconds) {
children foreach (_ ! Getter.Abort)
} ... }
```

What is the problem with above code?
It's not thread safe. The scheduler will run the code but it will not run in the context of actor, it will not run by the actor, but by scheduler. This means there is no protection this might run concurrently with the actor processing the next code. Both code might acess shared variable children, they try to modify and read from it. Could be unpredictable. 

How do we do this properly?
```scala
class Controller extends Actor with ActorLogging {
import context.dispatcher
var children = Set.empty[ActorRef]
context.system.scheduler.scheduleOnce(10.seconds, self, Timeout)
...
def receive = {
...
case Timeout => children foreach (_ ! Getter.Abort)
}
}
```
TSecond variant takes actor reference and message. The message will be delivered after the time elapsed ot the actor reference. In this we reiceive `TimeOut`, and we can abort children.
Similar issues can if you mix futures and actor.


```scala

```
