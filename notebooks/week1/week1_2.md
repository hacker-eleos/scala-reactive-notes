# Futures

We introduce high level abstraction for asynchronous programming called `Future`.

Remember the transformation we applied to a synchronous type signature
to make it asynchronous:

```scala
def program(a: A): B
def program(a: A, k: B => Unit): Unit
```
What if we could model an asynchronous result of type `T` as a return type `Future[T]`?

```scala
def program(a: A): Future[B]
```
This has the benifit of explicitly conveying `B` is a result as opposed to parameter.
```scala
def program(a: A, k: B => Unit): Unit
```
Let’s massage this type signature…

```scala
// by currying the continuation parameter
def program(a: A): (B => Unit) => Unit
```

```scala
// by introducing a type alias
type Future[+T] = (T => Unit) => Unit
def program(a: A): Future[B]
// bonus: adding failure handling
type Future[+T] = (Try[T] => Unit) => Unit
```

The standard library of scala provides a `Future` type however it's actual definition is slightly more sophisticated.
```scala
type Future[+T] = (Try[T] => Unit) => Unit
// by reifying the alias into a proper trait
trait Future[+T] extends ((Try[T] => Unit) => Unit) {
def apply(k: Try[T] => Unit): Unit
}
// by renaming ‘apply‘ to ‘onComplete‘
trait Future[+T] {
def onComplete(k: Try[T] => Unit): Unit
}
```
Let's revisit `coffeeBreak` with `Future`.

```scala
def makeCoffee(): Future[Coffee] = ...
def coffeeBreak(): Unit = {
makeCoffee().onComplete {
case Success(coffee) => drink(coffee)
case Failure(reason) => ...
}
chatWithColleagues()
}
```

* `onComplete` suffers from the same composability issues as callbacks
* `Future` provides convenient high-level transformation operations
(Simplified) API of Future:

```scala
trait Future[+A] {
def onComplete(k: Try[A] => Unit): Unit
// transform successful results
def map[B](f: A => B): Future[B]
def flatMap[B](f: A => Future[B]): Future[B]
def zip[B](fb: Future[B]): Future[(A, B)]
// transform failures
def recover(f: Exception => A): Future[A]
def recoverWith(f: Exception => Future[A]): Future[A]
}
```

## map Operation on Future

```scala
trait Future[+A] {
def map[B](f: A => B): Future[B]
}
```
* Transforms a successful `Future[A]` into a `Future[B]` by applying a function `f: A => B` after the `Future[A]` has completed
* Automatically propagates the failure of the former `Future[A]` (if any),to the resulting `Future[B]`

```scala
def grindBeans(): Future[GroundCoffee]
def brew(groundCoffee: GroundCoffee): Coffee
def makeCoffee(): Future[Coffee] =
grindBeans().map(groundCoffee => brew(groundCoffee))

```
For instance we have `grindBeans` operation returning `Future[GroundCoffee]` and `brew` operation turning `groundCoffee` into `Coffee`. We can make coffee by calling map on `grindBeans` and `brew`on the resulting `groundCoffee`.

This example is not realistic because `grindBeans` operation is asynchronous but `brew` operation is not. It instantly turns `groundCoffee` into `Coffee`. To make it realistic we should return `Future[Coffee]`. But if we do that the result type of `makeCoffee` would be `Future[Future[Coffee]]`.

This is why there exist `flatMap` operation.

## flatMap Operation on Future
```scala
trait Future[+A] {
def flatMap[B](f: A => Future[B]): Future[B]
}
```
* Transforms a successful `Future[A]` into a `Future[B]` by applying a function `f: A => Future[B]` after the `Future[A]` has completed
* Returns a failed `Future[B]` if the former `Future[A]` failed or if the `Future[B]` resulting from the application of the function `f` failed.

```scala
def grindBeans(): Future[GroundCoffee]
def brew(groundCoffee: GroundCoffee): Future[Coffee]
def makeCoffee(): Future[Coffee] =
grindBeans().flatMap(groundCoffee => brew(groundCoffee))
```

## zip Operation on Future
```scala
trait Future[+A] {
def zip[B](other: Future[B]): Future[(A, B)]
}
```
* Joins two successful `Future[A]` and `Future[B]` values into a single successful `Future[(A, B)]` value
* Returns a failure if any of the two `Future` values failed
* Does not create any dependency between the two Future values!

```scala
def makeTwoCoffees(): Future[(Coffee, Coffee)] =
makeCoffee() zip makeCoffee()
```
Here program `makeCoffee` evaluated concurrently two times. 

It is interesting to compare `zip` and `flatMap`. It is possible to asynchronously return pair of `Coffee`. But the key difference is second call to `makeCoffee` in `flatMap` necessarily  evaluated after the first coffee has been produced. In other `flatMap` introduces sequentiality.
```scala
def makeTwoCoffees(): Future[(Coffee, Coffee)] =
makeCoffee() zip makeCoffee()
def makeTwoCoffees(): Future[(Coffee, Coffee)] =
makeCoffee().flatMap { coffee1 =>
makeCoffee().map(coffee2 => (coffee1, coffee2))
}
```
Ofcourse the same behaviour can be implemented by moving the calls to `makeCoffee` to make outside `flatMap`. Here the calls to `makeCoffee` evaluted concurrently. 

```scala
def makeTwoCoffees(): Future[(Coffee, Coffee)] = {
val eventuallyCoffee1 = makeCoffee()
val eventuallyCoffee2 = makeCoffee()
eventuallyCoffee1.flatMap { coffee1 =>
eventuallyCoffee2.map(coffee2 => (coffee1, coffee2))
}
}
```
Only possible, when `makeCoffee` does not depend on the right hand side of `flatMap`.

## Sequencing Futures

If we chain mutliple `Future` with `flatMap`, we notice that like when we are using callbacks the order of computation follows the level of indentation. 

```scala
def work(): Future[Work] = ...
def coffeeBreak(): Future[Unit] = ...
def workRoutine(): Future[Work] = {
work().flatMap { work1 =>
coffeeBreak().flatMap { _ =>
work().map { work2 =>
work1 + work2
}
}
}
}
```
We can write above program using `for`-expression.
```scala
def work(): Future[Work] = ...
def coffeeBreak(): Future[Unit] = ...
def workRoutine(): Future[Work] =
for {
work1 <- work()
_ <- coffeeBreak()
work2 <- work()
} yield work1 + work2
```
If we use `for` expression, we get familiar top down order of computation.

```scala
def coffeeBreak(): Future[Unit] = {
val eventuallyCoffeeDrunk = makeCoffee().flatMap(drink)
val eventuallyChatted = chatWithColleagues()
eventuallyCoffeeDrunk.zip(eventuallyChatted)
.map(_ => ())
}
```
Instead of returning `Unit` we return `Future[Unit]`

## recover and recoverWith Operations on Future

Turn a failed Future into a successful one

```scala
trait Future[+A] {
def recover[B >: A](pf: PartialFunction[Throwable, B]): Future[B]
def recoverWith[B >: A](pf: PartialFunction[Throwable, Future[B]]): Future[B]
}
grindBeans()
.recoverWith { case BeansBucketEmpty =>
refillBeans().flatMap(_ => grindBeans())
}
.flatMap(coffeePowder => brew(coffeePowder))
```

## Execution Context

Where continuations are executed, physically?
When we call `onComplete` on `Future` where this continution is executed generally? It depends on the underlying system. In Scala the API of `Future` allow users to supply context of execution for continuations. 

User can choose

* Single thread (no parallelism)
* Thread Pool

In practice an execution context is passed via an implicit parameter, 

```scala
trait Future[+A] {
def onComplete(k: Try[A] => Unit)(implicit ec: ExecutionContext): Unit
}
import scala.concurrent.ExecutionContext.Implicits.global
```

A reasonable choice would be thread pool of size exactly as the underlying physical machine. This is exactly default execution context provided.



We have seen:
* The `Future[T]` type is an equivalent alternative to continuation passing
* Offers convenient transformation and failure recovering operations
* `map` and `flatMap` operations introduce sequentiality


```scala

```
