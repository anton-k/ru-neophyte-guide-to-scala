Part 15: Dealing With Failure in Actor Systems
=====================================================================

In the previous part of this series, I introduced you to the second cornerstone of Scala concurrency: The actor model, which complements the model based on composable futures backed by promises. You learnt how to define and create actors, how to send messages to them and how an actor processes these messages, possibly modifying its internal state as a result or asynchronously sending a response message to the sender.

While that was hopefully enough to get you interested in the actor model for concurrency, I left out some crucial concepts you will want to know about before starting to develop actor-based applications that consist of more than a simple echo actor.

The actor model is meant to help you achieve a high level of fault tolerance. In this article, we are going to have a look at how to deal with failure in an actor-based application, which is fundamentally different from error handling in a traditional layered server architecture.

The way you deal with failure is closely linked to some core Akka concepts and to some of the elements an actor system in Akka consists of. Hence, this article will also serve as a guide to those ideas and components.

Actor hierarchies
-----------------------------------------------------------------------

Before going into what happens when an error occurs in one of your actors, it’s essential to introduce one crucial idea underlying the actor approach to concurrency – an idea that is the very foundation for allowing you to build fault-tolerant concurrent applications: Actors are organized in a hierarchy.

So what does this mean? First of all, it means that every single of your actors has got a parent actor, and that each actor can create child actors. Basically, you can think of an actor system as a pyramid of actors. Parent actors watch over their children, just as in real life, taking care that they get back on their feet if they stumble. You will see shortly how exactly this is done.

### The guardian actor

In the previous article, we only had two different actors, a Barista actor and a Customer actor. I will not repeat their rather trivial implementations, but focus on how we created instances of these actor types:

~~~
import akka.actor.ActorSystem
val system = ActorSystem("Coffeehouse")
val barista = system.actorOf(Props[Barista], "Barista")
val customer = system.actorOf(Props(classOf[Customer], barista), "Customer")
~~~

As you can see, we create these two actors by calling the actorOf method defined on the ActorSystem type.

So what is the parent of these two actors? Is it the actor system? Not quite, but close. The actor system is not an actor itself, but it has got a so-called guardian actor that serves as the parent of all root-level user actors, i.e. actors we create by calling actorOf on our actor system.

There shouldn’t be a whole lot of actors in your system that are children of the guardian actor. It really makes more sense to have only a few top-level actors, each of them delegating most of the work to their children.

### Actor paths

The hierarchical structure of an actor system becomes apparent when looking at the actor paths of the actors you create. These are basically URLs by which actors can be addressed. You can get an actor’s path by calling path on its ActorRef:

~~~
barista.path // => akka.actor.ActorPath = akka://Coffeehouse/user/Barista
customer.path // => akka.actor.ActorPath = akka://Coffeehouse/user/Customer
~~~

The akka protocol is followed by the name of our actor system, the name of the user guardian actor and, finally, the name we gave our actor when calling actorOf on the system. In the case of remote actors, running on different machines, you would additionally see a host and a port.

Actor paths can be used to look up another actor. For example, instead of requiring the barista reference in its constructor, the Customer actor could call the actorSelection method on its ActorContext, passing in a relative path to retrieve a reference to the barista:

~~~
context.actorSelection("../Barista")
~~~

However, while being able to look up an actor by its path can sometimes come in handy, it’s often a much better idea to pass in dependencies in the constructor, just as we did before. Too much intimate knowledge about where your dependencies are located in the actor system makes your system more susceptible to bugs, and it will be difficult to refactor later on.

### An example hierarchy

To illustrate how parents watch over their children and what this has got to do with keeping your system fault-tolerant, I’m going to stick to the domain of the coffeehouse. Let’s give the Barista actor a child actor to which it can delegate some of the work involved in running a coffeehouse.

If we really were to model the work of a barista, we were likely giving them a bunch of child actors for all the various subtasks. But to keep this article focused, we have to be a little simplistic with our example.

Let’s assume that the barista has got a register. It processes transactions, printing appropriate receipts and incrementing the day’s sales so far. Here is a first version of it:

~~~
import akka.actor._
object Register {
  sealed trait Article
  case object Espresso extends Article
  case object Cappuccino extends Article
  case class Transaction(article: Article)
}
class Register extends Actor {
  import Register._
  import Barista._
  var revenue = 0
  val prices = Map[Article, Int](Espresso -> 150, Cappuccino -> 250)
  def receive = {
    case Transaction(article) =>
      val price = prices(article)
      sender ! createReceipt(price)
      revenue += price
  }
  def createReceipt(price: Int): Receipt = Receipt(price)
}
~~~

It contains an immutable map of the prices for each article, and an integer variable representing the revenue. Whenever it receives a Transaction message, it increments the revenue accordingly and returns a printed receipt to the sender.

The Register actor, as already mentioned, is supposed to be a child actor of the Barista actor, which means that we will not create it from our actor system, but from within our Barista actor. The initial version of our actor-come-parent looks like this:

~~~
object Barista {
  case object EspressoRequest
  case object ClosingTime
  case class EspressoCup(state: EspressoCup.State)
  object EspressoCup {
    sealed trait State
    case object Clean extends State
    case object Filled extends State
    case object Dirty extends State
  }
  case class Receipt(amount: Int)
}
class Barista extends Actor {
  import Barista._
  import Register._
  import EspressoCup._
  import context.dispatcher
  import akka.util.Timeout
  import akka.pattern.ask
  import akka.pattern.pipe
  import concurrent.duration._

  implicit val timeout = Timeout(4.seconds)
  val register = context.actorOf(Props[Register], "Register")
  def receive = {
    case EspressoRequest =>
      val receipt = register ? Transaction(Espresso)
      receipt.map((EspressoCup(Filled), _)).pipeTo(sender)
    case ClosingTime => context.stop(self)
  }
}
~~~

First off, we define the message types that our Barista actor is able to deal with. An EspressoCup can have one out of a fixed set of states, which we ensure by using a sealed trait.

The more interesting part is to be found in the implementation of the Barista class. The dispatcher, ask, and pipe imports as well as the implicit timeout are required because we make use of Akka’s ask syntax and futures in our Receive partial function: When we receive an EspressoRequest, we ask the Register actor for a Receipt for our Transaction. This is then combined with a filled espresso cup and piped to the sender, which will thus receive a tuple of type (EspressoCup, Receipt). This kind of delegating subtasks to child actors and then aggregating or amending their work is typical for actor-based applications.

Also, note how we create our child actor by calling actorOf on our ActorContext instead of the ActorSystem. By doing so, the actor we create becomes a child actor of the one who called this method, instead of a top-level actor whose parent is the guardian actor.

Finally, here is our Customer actor, which, like the Barista actor, will sit at the top level, just below the guardian actor:

~~~
object Customer {
  case object CaffeineWithdrawalWarning
}
class Customer(coffeeSource: ActorRef) extends Actor with ActorLogging {
  import Customer._
  import Barista._
  import EspressoCup._
  def receive = {
    case CaffeineWithdrawalWarning => coffeeSource ! EspressoRequest
    case (EspressoCup(Filled), Receipt(amount)) =>
      log.info(s"yay, caffeine for ${self}!")
  }
}
~~~

It is not terribly interesting for our tutorial, which focuses more on the Barista actor hierarchy. What’s new is the use of the ActorLogging trait, which allows us to write to the log instead of printing to the console.

Now, if we create our actor system and populate it with a Barista and two Customer actors, we can happily feed our two under-caffeinated addicts with a shot of black gold:

~~~
import Customer._
val system = ActorSystem("Coffeehouse")
val barista = system.actorOf(Props[Barista], "Barista")
val customerJohnny = system.actorOf(Props(classOf[Customer], barista), "Johnny")
val customerAlina = system.actorOf(Props(classOf[Customer], barista), "Alina")
customerJohnny ! CaffeineWithdrawalWarning
customerAlina ! CaffeineWithdrawalWarning
~~~

If you try this out, you should see two log messages from happy customers.


To crash or not to crash?
--------------------------------------------------------------

Of course, what we are really interested in, at least in this article, is not happy customers, but the question of what happens if things go wrong.

Our register is a fragile device – its printing functionality is not as reliable as it should be. Every so often, a paper jam causes it to fail. Let’s add a PaperJamException type to the Register companion object:

~~~
class PaperJamException(msg: String) extends Exception(msg)
~~~

Then, let’s change the createReceipt method in our Register actor accordingly:

~~~
def createReceipt(price: Int): Receipt = {
  import util.Random
  if (Random.nextBoolean())
    throw new PaperJamException("OMG, not again!")
  Receipt(price)
}
~~~

Now, when processing a Transaction message, our Register actor will throw a PaperJamException in about half of the cases.

What effect does this have on our actor system, or on our whole application? Luckily, Akka is very robust and not affected by exceptions in our code at all. What happens, though, is that the parent of the misbehaving child is notified – remember that parents are watching over their children, and this is the situation where they have to decide what to do.

### Supervisor strategies

The whole act of being notified about exceptions in child actors, however, is not handled by the parent actor’s Receive partial function, as that would confound the parent actor’s own behaviour with the logic for dealing with failure in its children. Instead, the two responsibilities are clearly separated.

Each actor defines its own supervisor strategy, which tells Akka how to deal with certain types of errors occurring in your children.

There are basically two different types of supervisor strategy, the OneForOneStrategy and the AllForOneStrategy. Choosing the former means that the way you want to deal with an error in one of your children will only affect the child actor from which the error originated, whereas the latter will affect all of your child actors. Which of those strategies is best depends a lot on your individual application.

Regardless of which type of SupervisorStrategy you choose for your actor, you will have to specify a Decider, which is a PartialFunction[Throwable, Directive] – this allows you to match against certain subtypes of Throwable and decide for each of them what’s supposed to happen to your problematic child actor (or all your child actors, if you chose the all-for-one strategy).

### Directives

Here is a list of the available directives:

~~~
sealed trait Directive
case object Resume extends Directive
case object Restart extends Directive
case object Stop extends Directive
case object Escalate extends Directive
~~~

* Resume: If you choose to Resume, this probably means that you think of your child actor as a little bit of a drama queen. You decide that the exception was not so exceptional after all – the child actor or actors will simply resume processing messages as if nothing extraordinary had happened.

* Restart: The Restart directive causes Akka to create a new instance of your child actor or actors. The reasoning behind this is that you assume that the internal state of the child/children is corrupted in some way so that it can no longer process any further messages. By restarting the actor, you hope to put it into a clean state again.

* Stop: You effectively kill the actor. It will not be restarted.

* Escalate: If you choose to Escalate, you probably don’t know how to deal with the failure at hand. You delegate the decision about what to do to your own parent actor, hoping they are wiser than you. If an actor escalates, they may very well be restarted themselves by their parent, as the parent will only decide about its own child actors.

### The default strategy

You don’t have to specify your own supervisor strategy in each and every actor. In fact, we haven’t done that so far. This means that the default supervisor strategy will take effect. It looks like this:

~~~
final val defaultStrategy: SupervisorStrategy = {
  def defaultDecider: Decider = {
    case _: ActorInitializationException ⇒ Stop
    case _: ActorKilledException         ⇒ Stop
    case _: Exception                    ⇒ Restart
  }
  OneForOneStrategy()(defaultDecider)
}
~~~

This means that for exceptions other than ActorInitializationException or ActorKilledException, the respective child actor in which the exception was thrown will be restarted.

Hence, when a PaperJamException occurs in our Register actor, the supervisor strategy of the parent actor (the barista) will cause the Register to be restarted, because we haven’t overridden the default strategy.

If you try this out, you will likely see an exception stacktrace in the log, but nothing about the Register actor being restarted.

Let’s verify that this is really happening. To do so, however, you will need to learn about the actor lifecycle.

### The actor lifecycle

To understand what the directives of a supervisor strategy actually do, it’s crucial to know a little bit about an actor’s lifecycle. Basically, it boils down to this: when created via actorOf, an actor is started. It can then be restarted an arbitrary number of times, in case there is a problem with it. Finally, an actor can be stopped, ultimately leading to its death.

There are numerous lifecycle hook methods that an actor implementation can override. It’s also important to know their default implementations. Let’s go through them briefly:

* preStart: Called when an actor is started, allowing you to do some initialization logic. The default implementation is empty.
* postStop: Empty by default, allowing you to clean up resources. Called after stop has been called for the actor.
* preRestart: Called right before a crashed actor is restarted. By default, it stops all children of that actor and then calls postStop to allow cleaning up of resources.
* postRestart: Called immediately after an actor has been restarted. Simply calls preStart by default.

This means that by default, restarting an actor entails a restart of its children. This may be exactly what you want, depending on your specific actor and use case. If it’s not what you want, these hook methods allow you to change that behaviour.

Let’s see if our Register gets indeed restarted upon failure by simply adding some log output to its postRestart method. Make the Register type extend the ActorLogging trait and add the following method to it:

~~~
override def postRestart(reason: Throwable) {
  super.postRestart(reason)
  log.info(s"Restarted because of ${reason.getMessage}")
}
~~~

Now, if you send the two Customer actors a bunch of CaffeineWithdrawalWarning messages, you should see the one or the other of those log outputs, confirming that our Register actor has been restarted.

### Death of an actor

Often, it doesn’t make sense to restart an actor again and again – think of an actor that talks to some other service over the network, and that service has been unreachable for a while. In such cases, it is a very good idea to tell Akka how often to restart an actor within a certain period of time. If that limit is exceeded, the actor is instead stopped and hence dies. Such a limit can be configured in the constructor of the supervisor strategy:

~~~
import scala.concurrent.duration._
import akka.actor.OneForOneStrategy
import akka.actor.SupervisorStrategy.Restart
OneForOneStrategy(10, 2.minutes) {
  case _ => Restart
}
~~~

### The self-healing system?

So, is our system running smoothly, healing itself whenever this damn paper jam occurs? Let’s change our log output:

~~~
override def postRestart(reason: Throwable) {
  super.postRestart(reason)
  log.info(s"Restarted, and revenue is $revenue cents")
}
~~~

And while we are at it, let’s also add some more logging to our Receive partial function, making it look like this:

~~~
def receive = {
  case Transaction(article) =>
    val price = prices(article)
    sender ! createReceipt(price)
    revenue += price
    log.info(s"Revenue incremented to $revenue cents")
}
~~~

Ouch! Something is clearly not as it should be. In the log, you will see the revenue increasing, but as soon as there is a paper jam and the Register actor restarts, it is reset to 0. This is because restarting indeed means that the old instance is discarded and a new one created as per the Props we initially passed to actorOf.

Of course, we could change our supervisor strategy, so that it resumes in case of a PaperJamException. We would have to add this to the Barista actor:

~~~
val decider: PartialFunction[Throwable, Directive] = {
  case _: PaperJamException => Resume
}
override def supervisorStrategy: SupervisorStrategy =
  OneForOneStrategy()(decider.orElse(SupervisorStrategy.defaultStrategy.decider))
~~~

Now, the actor is not restarted upon a PaperJamException, so its state is not reset.

### Error kernel

So we just found a nice solution to preserve the state of our Register actor, right?

Well, sometimes, simply resuming might be the best thing to do. But let’s assume that we really have to restart it, because otherwise the paper jam will not disappear. We can simulate this by maintaining a boolean flag that says if we are in a paper jam situation or not. Let’s change our Register like so:

~~~
class Register extends Actor with ActorLogging {
  import Register._
  import Barista._
  var revenue = 0
  val prices = Map[Article, Int](Espresso -> 150, Cappuccino -> 250)
  var paperJam = false
  override def postRestart(reason: Throwable) {
    super.postRestart(reason)
    log.info(s"Restarted, and revenue is $revenue cents")
  }
  def receive = {
    case Transaction(article) =>
      val price = prices(article)
      sender ! createReceipt(price)
      revenue += price
      log.info(s"Revenue incremented to $revenue cents")
  }
  def createReceipt(price: Int): Receipt = {
    import util.Random
    if (Random.nextBoolean()) paperJam = true
    if (paperJam) throw new PaperJamException("OMG, not again!")
    Receipt(price)
  }
}
~~~

Also remove the supervisor strategy we added to the Barista actor.

Now, the paper jam remains forever, until we have restarted the actor. Alas, we cannot do that without also losing important state regarding our revenue.

This is where the error kernel pattern comes in. Basically, it is just a simple guideline you should always try to follow, stating that if an actor carries important internal state, then it should delegate dangerous tasks to child actors, so as to prevent the state-carrying actor from crashing. Sometimes, it may make sense to spawn a new child actor for each such task, but that’s not a necessity.

The essence of the pattern is to keep important state as far at the top of the actor hierarchy as possible, while pushing error-prone tasks as far to the bottom of the hierarchy as possible.

Let’s apply this pattern to our Register actor. We will keep the revenue state in the Register actor, but move the error-prone behaviour of printing the receipt to a new child actor, which we appropriately enough call ReceiptPrinter. Here is the latter:

~~~
object ReceiptPrinter {
  case class PrintJob(amount: Int)
  class PaperJamException(msg: String) extends Exception(msg)
}
class ReceiptPrinter extends Actor with ActorLogging {
  var paperJam = false
  override def postRestart(reason: Throwable) {
    super.postRestart(reason)
    log.info(s"Restarted, paper jam == $paperJam")
  }
  def receive = {
    case PrintJob(amount) => sender ! createReceipt(amount)
  }
  def createReceipt(price: Int): Receipt = {
    if (Random.nextBoolean()) paperJam = true
    if (paperJam) throw new PaperJamException("OMG, not again!")
    Receipt(price)
  }
}
~~~

Again, we simulate the paper jam with a boolean flag and throw an exception each time someone asks us to print a receipt while in a paper jam. Other than the new message type, PrintJob, this is really just extracted from the Register type.

This is a good thing, not only because it moves away this dangerous operation from the stateful Register actor, but it also makes our code simpler and consequently easier to reason about: The ReceiptPrinter actor is responsible for exactly one thing, and the Register actor has become simpler, too, now being only responsible for managing the revenue, delegating the remaining functionality to a child actor:

~~~
class Register extends Actor with ActorLogging {
  import akka.pattern.ask
  import akka.pattern.pipe
  import context.dispatcher
  implicit val timeout = Timeout(4.seconds)
  var revenue = 0
  val prices = Map[Article, Int](Espresso -> 150, Cappuccino -> 250)
  val printer = context.actorOf(Props[ReceiptPrinter], "Printer")
  override def postRestart(reason: Throwable) {
    super.postRestart(reason)
    log.info(s"Restarted, and revenue is $revenue cents")
  }
  def receive = {
    case Transaction(article) =>
      val price = prices(article)
      val requester = sender
      (printer ? PrintJob(price)).map((requester, _)).pipeTo(self)
    case (requester: ActorRef, receipt: Receipt) =>
      revenue += receipt.amount
      log.info(s"revenue is $revenue cents")
      requester ! receipt
  }
}
~~~

We don’t spawn a new ReceiptPrinter for each Transaction message we get. Instead, we use the default supervisor strategy to have the printer actor restart upon failure.

One part that merits explanation is the weird way we increment our revenue: First we ask the printer for a receipt. We map the future to a tuple containing the answer as well as the requester, which is the sender of the Transaction message and pipe this to ourselves. When processing that message, we finally increment the revenue and send the receipt to the requester.

The reason for that indirection is that we want to make sure that we only increment our revenue if the receipt was successfully printed. Since it is vital to never ever modify the internal state of an actor inside of a future, we have to use this level of indirection. It helps us make sure that we only change the revenue within the confines of our actor, and not on some other thread.

Assigning the sender to a val is necessary for similar reasons: When mapping a future, we are no longer in the context of our actor either – since sender is a method, it would now likely return the reference to some other actor that has sent us a message, not the one we intended.

Now, our Register actor is safe from constantly being restarted, yay!

Of course, the very idea of having the printing of the receipt and the management of the revenue in one place is questionable. Having them together came in handy for demonstrating the error kernel pattern. Yet, it would certainly be a lot better to seperate the receipt printing from the revenue management altogether, as these are two concerns that don’t really belong together.

### Timeouts

Another thing that we may want to improve upon is the handling of timeouts. Currently, when an exception occurs in the ReceiptPrinter, this leads to an AskTimeoutException, which, since we are using the ask syntax, comes back to the Barista actor in an unsuccessfully completed Future.

Since the Barista actor simply maps over that future (which is success-biased) and then pipes the transformed result to the customer, the customer will also receive a Failure containing an AskTimeoutException.

The Customer didn’t ask for anything, though, so it is certainly not expecting such a message, and in fact, it currently doesn’t handle these messages. Let’s be friendly and send customers a ComebackLater message – this is a message they already understand, and it makes them try to get an espresso at a later point. This is clearly better, as the current solution means they will never know that they will not get their espresso.

To achieve this, let’s recover from AskTimeoutException failures by mapping them to ComebackLater messages. The Receive partial function of our Barista actor thus now looks like this:

~~~
def receive = {
  case EspressoRequest =>
    val receipt = register ? Transaction(Espresso)
    receipt.map((EspressoCup(Filled), _)).recover {
      case _: AskTimeoutException => ComebackLater
    } pipeTo(sender)
  case ClosingTime => context.system.shutdown()
}
~~~

Now, the Customer actors know they can try their luck later, and after trying often enough, they should finally get their eagerly anticipated espresso.

### Death Watch

Another principle that is important in order to keep your system fault-tolerant is to keep a watch on important dependencies – dependencies as opposed to children.

Sometimes, you have actors that depend on other actors without the latter being their children. This means that they can’t be their supervisors. Yet, it is important to keep a watch on their state and be notified if bad things happen.

Think, for instance, of an actor that is responsible for database access. You will want actors that require this actor to be alive and healthy to know when that is no longer the case. Maybe you want to switch your system to a maintenance mode in such a situation. For other use cases, simply using some kind of backup actor as a replacement for the dead actor may be a viable solution.

In any case, you will need to place a watch on an actor you depend on in order to get the sad news of its passing away. This is done by calling the watch method defined on ActorContext. To illustrate, let’s have our Customer actors watch the Barista – they are highly addicted to caffeine, so it’s fair to say they depend on the barista:

~~~
class Customer(coffeeSource: ActorRef) extends Actor with ActorLogging {
  import context.dispatcher

  context.watch(coffeeSource)

  def receive = {
    case CaffeineWithdrawalWarning => coffeeSource ! EspressoRequest
    case (EspressoCup(Filled), Receipt(amount)) =>
      log.info(s"yay, caffeine for ${self}!")
    case ComebackLater =>
      log.info("grumble, grumble")
      context.system.scheduler.scheduleOnce(300.millis) {
        coffeeSource ! EspressoRequest
      }
    case Terminated(barista) =>
      log.info("Oh well, let's find another coffeehouse...")
  }
}
~~~

We start watching our coffeeSource in our constructor, and we added a new case for messages of type Terminated – this is the kind of message we will receive from Akka if an actor we watch dies.

Now, if we send a ClosingTime to the message and the Barista tells its context to stop itself, the Customer actors will be notified. Give it a try, and you should see their output in the log.

Instead of simply logging that we are not amused, this could just as well initiate some failover logic, for instance.

Summary
--------------------------------------------------------

In this part of the series, which is the second one dealing with actors and Akka, you got to know some of the important components of an actor system, all while learning how to put the tools provided by Akka and the ideas behind it to use in order to make your system more fault-tolerant.

While there is still a lot more to learn about the actor model and Akka, we shall leave it at that for now, as this would go beyond the scope of this series. In the next part, which shall bring this series to a conclusion, I will point you to a bunch of Scala resources you may want to peruse to continue your journey through Scala land, and if actors and Akka got you excited, there will be something in there for you, too.

