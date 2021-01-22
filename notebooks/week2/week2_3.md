# Message Processing Semantics
## Actor Encapsulation

No direct access to actor behaviour is possible. Only possible by exchanging messages.

Messages can be send to known addresses(`ActorRef`)
- every actor knows its own address (self)
- creating an actor returns its address
- address can be sent within messages (e.g. `sender`)

It's not possible call methods on new created actors. Address can be sent within actors while sending messages.

Actors are completely independent agents of execution
- local execution, no notion of global syncrhonization. Every actor performs it's computation locally not shared directly with other actor (only by messages).
- all actors run fully concurrently.
- message passing primitive is one-way communication. When an actor sends a message it continues it's way.

One could say actors are completely isolated from each other. Actors are the most object oriented form of encapsulation. 

## Actor-Internal Evaluation Order
An actor is effectively single threaded.
- messages are recieved sequentially.
- behaviour change is effective before processing the next message.
- processing one message is the atomic unit of execution.

Everything an actor does for processing a single message cannot be interrupted or interleaved with processing of another message. This has the benifit of synchronized methods, but blocking is replaced by enqueing the messages.

## Bank Account - Revisited
It is good practice to define all actor messages (expected, returned, or replied) in a companion object

```scala
object BankAccount {
  case class Deposit(amount: BigInt) {
    require(amount > 0)
  }
  case class Withdraw(amount: BigInt) {
    require(amount > 0)
  }
  case object Done
  case object Failed
}

class BankAccount extends Actor {
  import BankAccount._
  var balance = BigInt(0)
  def receive = LoggingReceive {
    case Deposit(amount) =>
      balance += amount
      sender() ! Done
    case Withdraw(amount) if amount <= balance =>
      balance -= amount
      sender() ! Done
    case _ => sender() ! Failed
  }
}
    
```

This actor is equivalent to bank account defined earlier included syncrhonization. Code here will be serialzed. One message entered cannot be intereferd with other. Other messages will be forced to wait.

## Actor Collabration
- picture actors as persons
- model activities as actors

Suppose we have two accounts for Alice and Bob. Now we want to transfer money. We could include code to transfer money. But bankaccount should not know how to transfer. We introduce another actor Tom, who will be model of activity to transfer money. Tom will first send message to Alice to withdraw the money and Alice will reply success. In case of failure Tom will abort transaction. Tom will deposit Bob. Bob will reply success. Tom can possibly shutdown and inform some one else.

```scala
object WireTransfer {
  case class Transfer(from: ActorRef, to: ActorRef, amount: BigInt)
  case object Done
  case object Failed
}

class WireTransfer extends Actor {
  import WireTransfer._

  override def receive: Receive = LoggingReceive {
    case Transfer(from, to, amount) =>
      from ! BankAccount.Withdraw(amount)
      context.become(awaitFrom(to, amount, sender()))
  }

  def awaitFrom(to: ActorRef, amount: BigInt, customer: ActorRef): Receive = LoggingReceive {
    case BankAccount.Done =>
      to ! BankAccount.Deposit(amount)
      context.become(awaitTo(customer))
    case BankAccount.Failed =>
      customer ! Failed
      context.stop(self)
  }

  def awaitTo(customer: ActorRef): Receive = LoggingReceive {
    case BankAccount.Done =>
      customer ! Done
      context.stop(self)
  }
}
```

When `WireTransfer` receives `transfer(from, to, amount)` message it sends a `BankAccount.WithDraw(amount)` message to `from`. It further changes it behaviour to await the result of withdraw activity. It will suspend it's execution. It will not consume any resources, unless messages is sent to it.

Here is the main class 

```scala
class TransferMain extends Actor {
  val accountA = context.actorOf(Props[BankAccount], "accountA")
  val accountB = context.actorOf(Props[BankAccount], "accountB")

  accountA ! BankAccount.Deposit(100)

  def receive = LoggingReceive {
    case BankAccount.Done => transfer(150)
  }

  def transfer(amount: BigInt): Unit = {
    val transaction = context.actorOf(Props[WireTransfer], "transfer")
    transaction ! WireTransfer.Transfer(accountA, accountB, amount)
    context.become(LoggingReceive {
      case WireTransfer.Done =>
        println("success")
        context.stop(self)
      case WireTransfer.Failed =>
        println("failed")
        context.stop(self)
    })
  }
}
```


All actors interact using messages. Whenever you send you can never be sure it is recevied. The same is true for synchronous programs because for example the computer might crash. If you invoke net connection TCP/IP for transferring messages. Chance of message lost is high. Communication is unreliable. 

Successful delivery of message require eventual availability of channel and recipient. We can classify delivery and resulting gaurantee
* at-most-once: sending once $[0,1]$ times
* at-least-once: resending untill acknowledged delivers $[1, \infty]$
* exactly-once: processing only first reception deliver 1 time


## Reliable Messaging
Messages support reliability:
- all messages can be persisted.
- can include unique correlation IDs
- delivery can be retries until successful

Reliability can only be ensured by business-level acknowledgement. But one thing to note, it is not enough to deliver message to recipient unless you get a reply. All of these semantics only works if you include the processing actor replied saying I am done with specific task. Not enough to say I have put message in actor mailbox.

### How to make `transfer` reliable?
- log activities of `WireTransfer` to persistent storage
- each transfer has a unique ID
- add ID to withdraw and deposit.
- store ID of completed actions with `BankAccount`

## Message Ordering
If an actor sends multiple messages to the same destination, they will not arrive out of order (this is Akka-specific).



```scala

```
