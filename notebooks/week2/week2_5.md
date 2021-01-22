# Testing Actor Systems

Testing actors is integral to development. 

Test can only verify externally observable effects. Actors only interact through message passing there is no way to reach into them and to check their current behaviour without sending a message.

See the class 
```scala
class Toggle extends Actor{
  def happy: Receive = {
    case "How are you?" =>
      sender ! "happy"
      context become sad
  }
  def sad: Receive = {
    case "How are you?" =>
      sender ! "sad"
      context become happy
  }
  override def receive = happy
}

```
We must send "how are you?" messages. The context changes every time a message recevied. 

Akka `TestProbe()` is like a remote controlled actor. It's only purpose is to buffer incoming messages in internal queue so they can inspected in test procedure. When we write test, we cannot use `akka.Main` class we need to start the system. 

```scala
implicit val system: ActorSystem = ActorSystem("TestSys")
val toggle = system.actorOf(Props[Toggle])
val p = TestProbe()
p.send(toggle, "How are you?")
p.expectMsg("happy")
p.send(toggle, "How are you?")
p.expectMsg("sad")
p.send(toggle, "Unkown")
p.expectNoMessage(1.second)
system.stop()
```


The `ActorSystem` comes with so called gaurdian actor. `system.actorOf` create a request to gaurdian actor to create this actor for us. We also need to shut it down. The `TestProbe` is an actor driven from outside.  We can also create `TestProbe` inside. We can run a test in the context of probe. We can do it by using `TesKit` class.


```scala

new TestKit(ActorSystem("TestSys")) with ImplicitSender {
    val toggle = system.actorOf(Props[Toggle])
    send(toggle, "How are you?")
    expectMsg("happy")
    send(toggle, "How are you?")
    expectMsg("sad")
    send(toggle, "Unkown")
    expectNoMessage(1.second)
    system.stop()
}
```

Inside the class the `ActorSystem` is available with name `system`. The trait `ImplicitSender` will make internal small actor available implicitly so it will be picked up when you send messages. `toggle !"how are you?" testActor`. `expectMsg` is method on `TestKit`, so it is directly available here.

## Testing Actors with Dependencies

Some Actor might have external dependencies. For example the need to talk to database, or web service. Traditional solution is to use dependecny injection. You can use Akka together with Spring. One simple solution is to add overridable factory methods.

Let's look at `Receptionist`. 


```scala

```




```scala

```


```scala

```
