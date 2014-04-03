Part 8: Welcome to the Future
====================================================

As an aspiring and enthusiastic Scala developer, you will likely have heard of Scala’s approach at dealing with concurrency – or maybe that was even what attracted you in the first place. Said approach makes reasoning about concurrency and writing well-behaved concurrent programs a lot easier than the rather low-level concurrency APIs you are confronted with in most other languages.

One of the two cornerstones of this approach is the Future, the other being the Actor. The former shall be the subject of this article. I will explain what futures are good for and how you can make use of them in a functional way.

Please make sure that you have version 2.9.3 or later if you want to get your hands dirty and try out the examples yourself. The futures we are discussing here were only incorporated into the Scala core distribution with the 2.10.0 release and later backported to Scala 2.9.3. Originally, with a slightly different API, they were part of the Akka concurrency toolkit.


Why sequential code can be bad
------------------------------------------------------

Suppose you want to prepare a cappuccino. You could simply execute the following steps, one after another:

1. Grind the required coffee beans
2. Heat some water
3. Brew an espresso using the ground coffee and the heated water
4. Froth some milk
5. Combine the espresso and the frothed milk to a cappuccino

Translated to Scala code, you would do something like this:

~~~
import scala.util.Try
// Some type aliases, just for getting more meaningful method signatures:
type CoffeeBeans = String
type GroundCoffee = String
case class Water(temperature: Int)
type Milk = String
type FrothedMilk = String
type Espresso = String
type Cappuccino = String
// dummy implementations of the individual steps:
def grind(beans: CoffeeBeans): GroundCoffee = s"ground coffee of $beans"
def heatWater(water: Water): Water = water.copy(temperature = 85)
def frothMilk(milk: Milk): FrothedMilk = s"frothed $milk"
def brew(coffee: GroundCoffee, heatedWater: Water): Espresso = "espresso"
def combine(espresso: Espresso, frothedMilk: FrothedMilk): Cappuccino = "cappuccino"
// some exceptions for things that might go wrong in the individual steps
// (we'll need some of them later, use the others when experimenting
// with the code):
case class GrindingException(msg: String) extends Exception(msg)
case class FrothingException(msg: String) extends Exception(msg)
case class WaterBoilingException(msg: String) extends Exception(msg)
case class BrewingException(msg: String) extends Exception(msg)
// going through these steps sequentially:
def prepareCappuccino(): Try[Cappuccino] = for {
  ground <- Try(grind("arabica beans"))
  water <- Try(heatWater(Water(25)))
  espresso <- Try(brew(ground, water))
  foam <- Try(frothMilk("milk"))
} yield combine(espresso, foam)
~~~

Doing it like this has several advantages: You get a very readable step-by-step instruction of what to do. Moreover, you will likely not get confused while preparing the cappuccino this way, since you are avoiding context switches.

On the downside, preparing your cappuccino in such a step-by-step manner means that your brain and body are on wait during large parts of the whole process. While waiting for the ground coffee, you are effectively blocked. Only when that’s finished, you’re able to start heating some water, and so on.

This is clearly a waste of valuable resources. It’s very likely that you would want to initiate multiple steps and have them execute concurrently. Once you see that the water and the ground coffee is ready, you’d start brewing the espresso, in the meantime already starting the process of frothing the milk.

It’s really no different when writing a piece of software. A web server only has so many threads for processing requests and creating appropriate responses. You don’t want to block these valuable threads by waiting for the results of a database query or a call to another HTTP service. Instead, you want an asynchronous programming model and non-blocking IO, so that, while the processing of one request is waiting for the response from a database, the web server thread handling that request can serve the needs of some other request instead of idling along.

> “I heard you like callbacks, so I put a callback in your callback!”

Of course, you already knew all that - what with Node.js being all the rage among the cool kids for a while now. The approach used by Node.js and some others is to communicate via callbacks, exclusively. Unfortunately, this can very easily lead to a convoluted mess of callbacks within callbacks within callbacks, making your code hard to read and debug.

Scala’s Future allows callbacks, too, as you will see very shortly, but it provides much better alternatives, so it’s likely you won’t need them a lot.

> “I know Futures, and they are completely useless!”

You might also be familiar with other Future implementations, most notably the one provided by Java. There is not really much you can do with a Java future other than checking if it’s completed or simply blocking until it is completed. In short, they are nearly useless and definitely not a joy to work with.

If you think that Scala’s futures are anything like that, get ready for a surprise. Here we go!

Semantics of Future
----------------------------------------

Scala’s Future[T], residing in the scala.concurrent package, is a container type, representing a computation that is supposed to eventually result in a value of type T. Alas, the computation might go wrong or time out, so when the future is completed, it may not have been successful after all, in which case it contains an exception instead.

Future is a write-once container – after a future has been completed, it is effectively immutable. Also, the Future type only provides an interface for reading the value to be computed. The task of writing the computed value is achieved via a Promise. Hence, there is a clear separation of concerns in the API design. In this post, we are focussing on the former, postponing the use of the Promise type to the next article in this series.

Working with Futures
---------------------------------------

There are several ways you can work with Scala futures, which we are going to examine by rewriting our cappuccino example to make use of the Future type. First, we need to rewrite all of the functions that can be executed concurrently so that they immediately return a Future instead of computing their result in a blocking way:

~~~
import scala.concurrent.future
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.util.Random

def grind(beans: CoffeeBeans): Future[GroundCoffee] = Future {
  println("start grinding...")
  Thread.sleep(Random.nextInt(2000))
  if (beans == "baked beans") throw GrindingException("are you joking?")
  println("finished grinding...")
  s"ground coffee of $beans"
}

def heatWater(water: Water): Future[Water] = Future {
  println("heating the water now")
  Thread.sleep(Random.nextInt(2000))
  println("hot, it's hot!")
  water.copy(temperature = 85)
}

def frothMilk(milk: Milk): Future[FrothedMilk] = Future {
  println("milk frothing system engaged!")
  Thread.sleep(Random.nextInt(2000))
  println("shutting down milk frothing system")
  s"frothed $milk"
}

def brew(coffee: GroundCoffee, heatedWater: Water): Future[Espresso] = Future {
  println("happy brewing :)")
  Thread.sleep(Random.nextInt(2000))
  println("it's brewed!")
  "espresso"
}
~~~

There are several things that require an explanation here.

First off, there is the apply method on the Future companion object, that requires two arguments:

~~~
object Future {
  def apply[T](body: => T)(implicit execctx: ExecutionContext): Future[T]
}
~~~

The computation to be computed asynchronously is passed in as the body by-name parameter. The second argument, in its own argument list, is an implicit one, which means we don’t have to specify one if a matching implicit value is defined somewhere in scope. We make sure this is the case by importing the global execution context.

An ExecutionContext is something that can execute our future, and you can think of it as something like a thread pool. Since the ExecutionContext is available implicitly, we only have a single one-element argument list remaining. Single-argument lists can be enclosed with curly braces instead of parentheses. People often make use of this when calling the future method, making it look a little bit like we are using a feature of the language and not calling an ordinary method. The ExecutionContext is an implicit parameter for virtually all of the Future API.

Furthermore, of course, in this simple example, we don’t actually compute anything, which is why we are putting in some random sleep, simply for demonstration purposes. We also print to the console before and after our “computation” to make the non-deterministic and concurrent nature of our code clearer when trying it out.

The computation of the value to be returned by a Future will start at some non-deterministic time after that Future instance has been created, by some thread assigned to it by the ExecutionContext.

Callbacks
--------------------------

Sometimes, when things are simple, using a callback can be perfectly fine. Callbacks for futures are partial functions. You can pass a callback to the onSuccess method. It will only be called if the Future completes successfully, and if so, it receives the computed value as its input:

~~~
grind("arabica beans").onSuccess { case ground =>
  println("okay, got my ground coffee")
}
~~~

Similarly, you could register a failure callback with the onFailure method. Your callback will receive a Throwable, but it will only be called if the Future did not complete successfully.

Usually, it’s better to combine these two and register a completion callback that will handle both cases. The input parameter for that callback is a Try:

~~~
import scala.util.{Success, Failure}
grind("baked beans").onComplete {
  case Success(ground) => println(s"got my $ground")
  case Failure(ex) => println("This grinder needs a replacement, seriously!")
}
~~~

Since we are passing in baked beans, an exception occurs in the grind method, leading to the Future completing with a Failure.

Composing futures
------------------------------------------

Using callbacks can be quite painful if you have to start nesting callbacks. Thankfully, you don’t have to do that! The real power of the Scala futures is that they are composable.

If you have followed this series, you will have noticed that all the container types we discussed made it possible for you to map them, flat map them, or use them in for comprehensions and that I mentioned that Future is a container type, too. Hence, the fact that Scala’s Future type allows you to do all that will not come as a surprise at all.

The real question is: What does it really mean to perform these operations on something that hasn’t even finished computing yet?

### Mapping the future

Haven’t you always wanted to be a traveller in time who sets out to map the future? As a Scala developer you can do exactly that! Suppose that once your water has heated you want to check if its temperature is okay. You can do so by mapping your Future[Water] to a Future[Boolean]:

~~~
val temperatureOkay: Future[Boolean] = heatWater(Water(25)).map { water =>
  println("we're in the future!")
  (80 to 85).contains(water.temperature)
}
~~~

The Future[Boolean] assigned to temperatureOkay will eventually contain the successfully computed boolean value. Go change the implementation of heatWater so that it throws an exception (maybe because your water heater explodes or something) and watch how we're in the future will never be printed to the console.

When you are writing the function you pass to map, you’re in the future, or rather in a possible future. That mapping function gets executed as soon as your Future[Water] instance has completed successfully. However, the timeline in which that happens might not be the one you live in. If your instance of Future[Water] fails, what’s taking place in the function you passed to map will never happen. Instead, the result of calling map will be a Future[Boolean] containing a Failure.

### Keeping the future flat

If the computation of one Future depends on the result of another, you’ll likely want to resort to flatMap to avoid a deeply nested structure of futures.

For example, let’s assume that the process of actually measuring the temperature takes a while, so you want to determine whether the temperature is okay asynchronously, too. You have a function that takes an instance of Water and returns a Future[Boolean]:

~~~
def temperatureOkay(water: Water): Future[Boolean] = Future {
  (80 to 85).contains(water.temperature)
}
~~~

Use flatMap instead of map in order to get a Future[Boolean] instead of a Future[Future[Boolean]]:

~~~
val nestedFuture: Future[Future[Boolean]] = heatWater(Water(25)).map {
  water => temperatureOkay(water)
}
val flatFuture: Future[Boolean] = heatWater(Water(25)).flatMap {
  water => temperatureOkay(water)
}
~~~

Again, the mapping function is only executed after (and if) the Future[Water] instance has been completed successfully, hopefully with an acceptable temperature.

### For comprehensions

Instead of calling flatMap, you’ll usually want to write a for comprehension, which is essentially the same, but reads a lot clearer. Our example above could be rewritten like this:

~~~
val acceptable: Future[Boolean] = for {
  heatedWater <- heatWater(Water(25))
  okay <- temperatureOkay(heatedWater)
} yield okay
~~~

If you have multiple computations that can be computed in parallel, you need to take care that you already create the corresponding Future instances outside of the for comprehension.

~~~
def prepareCappuccinoSequentially(): Future[Cappuccino] = {
  for {
    ground <- grind("arabica beans")
    water <- heatWater(Water(20))
    foam <- frothMilk("milk")
    espresso <- brew(ground, water)
  } yield combine(espresso, foam)
}
~~~

This reads nicely, but since a for comprehension is just another representation for nested flatMap calls, this means that the Future[Water] created in heatWater is only really instantiated after the Future[GroundCoffee] has completed successfully. You can check this by watching the sequential console output coming from the functions we implemented above.

Hence, make sure to instantiate all your independent futures before the for comprehension:

~~~
def prepareCappuccino(): Future[Cappuccino] = {
  val groundCoffee = grind("arabica beans")
  val heatedWater = heatWater(Water(20))
  val frothedMilk = frothMilk("milk")
  for {
    ground <- groundCoffee
    water <- heatedWater
    foam <- frothedMilk
    espresso <- brew(ground, water)
  } yield combine(espresso, foam)
}
~~~

Now, the three futures we create before the for comprehension start being completed immediately and execute concurrently. If you watch the console output, you will see that it’s non-deterministic. The only thing that’s certain is that the "happy brewing" output will come last. Since the method in which it is called requires the values coming from two other futures, it is only created inside our for comprehension, i.e. after those futures have completed successfully.

### Failure projections

You will have noticed that Future[T] is success-biased, allowing you to use map, flatMap, filter etc. under the assumption that it will complete successfully. Sometimes, you may want to be able to work in this nice functional way for the timeline in which things go wrong. By calling the failed method on an instance of Future[T], you get a failure projection of it, which is a Future[Throwable]. Now you can map that Future[Throwable], for example, and your mapping function will only be executed if the original Future[T] has completed with a failure.

Outlook
------------------------------------------------

You have seen the Future, and it looks bright! The fact that it’s just another container type that can be composed and used in a functional way makes working with it very pleasant.

Making blocking code concurrent can be pretty easy by wrapping it in a call to future. However, it’s better to be non-blocking in the first place. To achieve this, one has to make a Promise to complete a Future. This and using the futures in practice will be the topic of the next part of this series.


