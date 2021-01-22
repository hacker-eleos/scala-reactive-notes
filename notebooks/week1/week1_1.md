# Asynchronous Programming

* Execution of a computation on another computing unit, without waiting for its termination ;
* Better resource efficiency

Since the execution of asynchronous program is concurent, how can we say that some computation must be executed after another computation is finished?



What if a program A depends on the result of an asynchronously executed program B?
```scala
def coffeeBreak(): Unit = {
val coffee = makeCoffee()
drink(coffee)
chatWithColleagues()
}

```

Here if the call to `makeCoffee` is asynchronous, the execution of `makeCoffee` would happen concurently with remaining of the execution of the `coffeeBreak`. It means that we might try to `drink` without `makeCoffee` is finished.

How can we make this synchronous call into asynchronous but still controlling the order in which the computation executed. 

Simplest way is to use callbacks. 

Asynchronous version of `makeCoffee` with following type signature.

```scala
def makeCoffee(coffeeDone: Coffee => Unit): Unit = {
// work hard ...
// ... and eventually
val coffee = ...
coffeeDone(coffee)
}
def coffeeBreak(): Unit = {
makeCoffee { coffee =>
drink(coffee)
}
chatWithColleagues()
}
```

In this version `makeCoffee` takes a function which calls it when coffee is done. In `coffeeBreak` coffee is taken after it's produced.

A synchronous type signature can be turned into an asynchronous type
signature by:
* returning `Unit`
* and taking as parameter a continuation defining what to do after the return value has been computed

## Combining Asynchronous Programs

```scala
def makeCoffee(coffeeDone: Coffee => Unit): Unit = ...
def makeTwoCoffees(coffeesDone: (Coffee, Coffee) => Unit): Unit = {
var firstCoffee: Option[Coffee] = None
val k = { coffee: Coffee =>
firstCoffee match {
case None => firstCoffee = Some(coffee)
case Some(coffee2) => coffeesDone(coffee, coffee2)
}
}
makeCoffee(k)
makeCoffee(k)
}
```

We call `makeCoffee` two times but we don't know which call will be finished first, so which call back will be called first. So, in the continutation we check the coffee that has been produced is the first one the two. If that's the case we can save that coffee, otherwise we call the callback with the two produced coffee.

The fact that `makeCoffee` callback returns `unit` forces us to use `var firstCofee` variable.
This program style is error prone, because it's mutable.


What if another program depends on the coffee break to be done?

```scala
def coffeeBreak(): Unit = ...
```
* We need to make coffeeBreak take a callback too!

```scala
def coffeeBreak(breakDone: Unit => Unit): Unit = ...
def workRoutine(workDone: Work => Unit): Unit = {
work { work1 =>
coffeeBreak { _ =>
work { work2 =>
workDone(work1 + work2)
}
}
}
}
```
* Order of execution follows the indentation level!

Handling Failures

* In synchronous programs, failures are handled with exceptions ;
* What happens if an asynchronous call fails?
    * We need a way to propagate the failure to the call site


```scala
def makeCoffee(coffeeDone: Try[Coffee] => Unit): Unit = ...
```

## Summary

In this video, we have seen:

* How to sequence asynchronous computations using callbacks
* Callbacks introduce complex type signatures
* The continuation passing style is tedious to use


```scala

```
