# Actor Model

## What is an Actor?
The Actor Model represents object and their interactions, resembling human organizations and laws of physics.

An Actor
- is an object with identity
- has a behaviour
- and interacts only with _asynchronous_ message passing.




## The Actor trait

```scala

type Receive = PartialFunction[Any, Unit]
trait Actor{
    def receive: Receive
    ...
}
```
In Akka, Actor trait defines one abstract method `receive`, and returns type `Recieve`. `Recieve` is a partial function from `Any` to `Unit`. It describes the response of actor to message. Any message can come in so the type `Any` and it doesn't return anything because caller is long gone due to asynchronous message passing.

### A simple Actor
```scala
class Counter extends Actor{
    var count = 0
    def receive = {
        case "incr" => count + 1
    }
}
```
 If we get message string "incr" then increment `count`. This object doesn't exhibit stateful behaviour, because we can only send one string and cannot get any answer back.
 
 If we want to make it stateful, we shoould enable other actors to let them find out the value of count.
 
 
## Stateful Actor

Actors can send messages to addresses they know. Address are modeled by Akka as `ActorRef`

### A simple Actor
```scala
class Counter extends Actor{
    var count = 0
    def receive = {
        case "incr" => count + 1
        case ("get", customer: ActorRef) => customer ! count
    }
}
```

We add another case. If we get a tuple with string "get" and something of type `ActorRef`, then we can send **to the customer** the message count. `!` is called tell and used to send messages in Akka. customer tell count.

## How messages are sent?

```scala
trait Actor{
implicit val self: ActorRef
def sender: ActorRef
...
}
```

Each Actor knows it's address. It's called self and it's implicitly available.

```scala
abstract class ActorRef{
    def !(msg: Any)(implicit sender: ActorRef = Actor.noSender): Unit
    def tell(msg: Any, sender: ActorRef) = this.!(msg)(sender)

```
`ActorRef` is an abstract class. If you use `!` operator within an Actor it will implicitly pickup sender as being the self refernce of the Actor. 

Within the receiving Actor this value is available as `sender` which gives the `ActorRef` of Actor which has sent message currently being processed. Passing the sender reference along with the message to the receving Actor needs to be done explicitly in languages like Erlang, but it such a common pattern, we devoted this one name to Actor and do it automatically.


Using the above power we can rewrite
```scala
class Counter extends Actor{
    var count = 0
    def receive = {
        case "incr" => count + 1
        case "get" => sender ! count
    }
}
```

Sender is automatically picked up to which the reply is sent.

## Actor's Context
An Actor can create more Actors and change it's behaviour. The Actor type describe behaviour but the execution machinery is provided by `ActorContext`.
```scala

trait ActorContext {
    def become(behaviour: Receive, discardOld: Boolean = True): Unit
    def unbecome(): Unit
}
trait Actor{
    implicit val context: ActorContext
}
```

Each Actor has stack of behaviours. The top one is always the active one. Default mode of `become` is to replace top most value, but you can also push and `unbecome` to become top behaviour. You have access to context within the Actor by just saying `context`. Let's see some action.

```scala
class Counter extends Actor{
    def counter(n: Int): Receive = {
        case "incr" => context.become(counter(n+1))
        case "get" => sender ! n
    }
    def receive = counter(0)
}
```
First we need to define a method which gives us behaviour(`Receive`). This method `counter` takes an argument state of the counter currently is. We start a counter at 0. If we get "incr" we change behaviour of that `counter(1)`, within `counter` behaviour, we can get request we can reply with current counter value.

We can see it looks bit like tail recursive function, we call it inside itself, but it is asynchronous. `context.become` evaluates when next message is processed.

First of all there is only place where state is changed, it is `become` method. State is scoped to the current behaviour. 

## Create and stop Actors
```scala

trait ActorContext{
    def actorOf(p: Props, name: String): ActorRef
    def stop(a: ActorRef): Unit
}
```

`actorOf` takes `Props`, a description on how to create an Actor and a name and it returns `ActorRef`. Each Actor is created by another Actor and exactly one Actor. So Actors follow a heierachy. `stop` is often applied to `self`.
Let's try out.
```scala
class Main extends Actor{
    val counter = context.ofActor(Props[Counter], "counter")
    counter ! "get"
    counter ! "incr"
    counter ! "incr"
    counter ! "incr"
    counter ! "get"
    
    def receive = {
        case count: Int => println(s"count was $count")
        context.stop(self)
        }
    }
    

```


```scala

```
