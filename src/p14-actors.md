Глава 14: Паралельные вычисления  с помощью акторов
==============================================================

После нескольких статей о продвинутых возможностях системы типов Scala и как
они могут быть использованы для повышения гибкости кода и стабильности за
счёт объявления более жёстких ограничений на этапе компиляции, мы вернёмся
к вопросу о параллельных вычислениях.  

В прошлых статьях мы узнали как организовать асинхронные вычисления с помощью `Future`.

Этот способ очень хорошо подходит для решения многих проблем. Но есть и другие альтернативы. 
Модель акторов -- это другой основополагающих подход к распараллеливанию программ. В этом
подходе одновременно выполняющиеся процессы передают друг другу сообщения. 

Идея акторов не нова. Наиболее выдающаяся реализация было сделана в языке Erlang. 
В стандартной поставке Scala также есть своя библиотека для реализации акторов,
но она устарела и с версии 2.11 будет окончательно заменена на реализацию акторов
из библиотеки Akka. Эта библиотека по-сути уже давно является стандартом. 

В этой статье мы познакомимся с моделью акторов и узнаем основы применения 
акторов в библиотеке Akka. Это лишь поверхностное знакомство, в этом настоящая статья
отличается от предыдущих, мы сосредоточимся на том, чтобы научить думать в
терминах Akka, изучим достаточно материала для того чтобы Вы могли заинтересоваться
этой замечательной библиотекой. 


Проблема общего изменяемого состояния
-------------------------------------------------------------------

На сегодняшний день доминирующий подходом к созданию параллельных программ является
подход, основанные на общем изменяемом сотоянии (shared mutable state) -- большое
чило объектов с состоянием выполняются в своих потоках, состояние каждого объекта 
может быть изменено сразу в нескольких частях. Обычно код такого приложения
насыщен семафорами, контролирующими чтение и запись, для того чтобы состояние изменялось атомарно,
для избежания одновременного изменения состояния несколькими потоками. 
В то же время мы стараемся не окружать семафорами слишком большие куски кода,
поскольку это приводит к существенному снижению скорости приложения.

Очень часто такой код с самого начала пишется без учёта возможности распараллеливания.
Параллелизация происходит по необходимости. Когда мы начинаем с последовательного
кода всё идёт гладко, но когда мы пытаемся распараллелить его  с учётом всех потребностей
вычислений в многопоточной среде, мы получаем код, крайне трудный для чтения, который
очень сложно понять. 

Корень проблемы в том, что такие низкоуровневые средства синхронизации вычислений
как семафоры и потоки -- очень сложны для понимания. А следовательно, в их использовании
очень легко ошибиться. Если вы не понимаете, что происходит в вашей системе, не сомневайтесь
рано или поздно в ней появятся трудно уловимые ошибки, связанные с взаимной блокировкой процессов,
или просто что-то может пойти не так. Такие ошибки могут начать проявляться после месяцев нормальной 
работы серверов, когда вы давно установили его на боевые машины. 

Также с этими низкоуровневыми конструкциями очень сложно добиться приемлемой эффективности.


Модель акторов
---------------------------------------------------------------------

The Actor programming model is aimed at avoiding all the problems described above, allowing you to write highly performant concurrent code that is easy to reason about. Unlike the widely used approach of shared mutable state, it requires you to design and write your application from the ground up with concurrency in mind – it’s not really possible to add support for it later on.

The idea is that your application consists of lots of light-weight entities called actors. Each of these actors is responsible for only a very small task, and is thus easy to reason about. A more complex business logic arises out of the interaction between several actors, delegating tasks to others or passing messages to collaborators for other reasons.

### The Actor System

Actors are pitiful creatures: They cannot live on their own. Rather, each and every actor in Akka resides in and is created by an actor system. Aside from allowing you to create and find actors, an ActorSystem provides for a whole bunch of additional functionality, none of which shall concern us right now.

In order to try out the example code, please add the following resolver and dependency to your SBT-based Scala 2.10 project first:

~~~
resolvers += "Typesafe Releases" at "http://repo.typesafe.com/typesafe/releases"

libraryDependencies += "com.typesafe.akka" %% "akka-actor" % "2.2.3"
~~~

Now, let’s create an ActorSystem. We’ll need it as an environment for our actors:

~~~
import akka.actor.ActorSystem
object Barista extends App {
  val system = ActorSystem("Barista")
  system.shutdown()
}
~~~

We created a new instance of ActorSystem and gave it the name "Barista" – we are returning to the domain of coffee, which should be familiar from the article on composable futures.

Finally, we are good citizens and shut down our actor system once we no longer need it.

### Defining an actor

Whether your application consists of a few dozen or a few million actors totally depends on your use case, but Akka is absolutely okay with a few million. You might be baffled by this insanely high number. It’s important to understand that there is not a one-to-one relationship between an actor and a thread. You would soon run out of memory if that were the case. Rather, due to the non-blocking nature of actors, one thread can execute many actors – switching between them depending on which of them has messages to be processed.

To understand what is actually happening, let’s first create a very simple actor, a Barista that can receive orders but doesn’t really do anything apart from printing messages to the console:

~~~
sealed trait CoffeeRequest
case object CappuccinoRequest extends CoffeeRequest
case object EspressoRequest extends CoffeeRequest

import akka.actor.Actor
class Barista extends Actor {
  def receive = {
    case CappuccinoRequest => println("I have to prepare a cappuccino!")
    case EspressoRequest => println("Let's prepare an espresso.")
  }
}
~~~

First, we define the types of messages that our actor understands. Typically, case classes are used for messages sent between actors if you need to pass along any parameters. If all the actor needs is an unparameterized message, this message is typically represented as a case object – which is exactly what we are doing here.

In any case, it’s crucial that your messages are immutable, or else bad things will happen.

Next, let’s have a look at our class Barista, which is the actual actor, extending the aptly named Actor trait. Said trait defines a method receive which returns a value of type Receive. The latter is really only a type alias for PartialFunction[Any, Unit].

### Processing messages

So what’s the meaning of this receive method? The return type, PartialFunction[Any, Unit] may seem strange to you in more than one respect.

In a nutshell, the partial function returned by the receive method is responsible for processing your messages. Whenever another part of your software – be it another actor or not – sends your actor a message, Akka will eventually let it process this message by calling the partial function returned by your actor’s receive method, passing it the message as an argument.

#### Side-effecting

When processing a message, an actor can do whatever you want it to, apart from returning a value.

Wat!?

As the return type of Unit suggests, your partial function is side-effecting. This might come as a bit of a shock to you after we emphasized the usage of pure functions all the time. For a concurrent programming model, this actually makes a lot of sense. Actors are where your state is located, and having some clearly defined places where side-effects will occur in a controllable manner is totally fine – each message your actor receives is processed in isolation, one after another, so there is no need to reason about synchronization or locks.

#### Untyped

But… this partial function is not only side-effecting, it’s also as untyped as you can get in Scala, expecting an argument of type Any. Why is that, when we have such a powerful type system at our fingertips?

This has a lot to do with some important design choices in Akka that allow you to do things like forwarding messages to other actors, installing load balancing or proxying actors without the sender having to know anything about them and so on.

In practice, this is usually not a problem. With the messages themselves being strongly typed, you typically use pattern matching for processing those types of messages you are interested in, just as we did in our tiny example above.

Sometimes though, the weakly typed actors can indeed lead to nasty bugs the compiler can’t catch for you. If you have grown to love the benefits of a strong type system and think you don’t want to go away from that at any costs for some parts of your application, you may want to look at Akka’s new experimental Typed Channels feature.

#### Asynchronous and non-blocking

I wrote above that Akka would let your actor eventually process a message sent to it. This is important to keep in mind: Sending a message and processing it is done in an asynchronous and non-blocking fashion. The sender will not be blocked until the message has been processed by the receiver. Instead, they can immediately continue with their own work. Maybe they expect to get a messsage from your actor in return after a while, or maybe they are not interested in hearing back from your actor at all.

What really happens when some component sends a message to an actor is that this message is delivered to the actor’s mailbox, which is basically a queue. Placing a message in an actor’s mailbox is a non-blocking operation, i.e. the sender doesn’t have to wait until the message is actually enqueued in the recipient’s mailbox.

The dispatcher will notice the arrival of a new message in an actor’s mailbox, again asynchronously. If the actor is not already processing a previous message, it is now allocated to one of the threads available in the execution context. Once the actor is done processing any previous messages, the dispatcher sends it the next message from its mailbox for processing.

The actor blocks the thread to which it is allocated for as long as it takes to process the message. While this doesn’t block the sender of the message, it means that lengthy operations degrade overall performance, as all the other actors have to be scheduled for processing messages on one of the remaining threads.

Hence, a core principle to follow for your Receive partial functions is to spend as little time inside them as possible. Most importantly, avoid calling blocking code inside your message processing code, if possible at all.

Of course, this is something you can’t prevent doing completely – the majority of database drivers nowadays is still blocking, and you will want to be able to persist data or query for it from your actor-based application. There are solutions to this dilemma, but we won’t cover them in this introductory article.

### Creating an actor

Defining an actor is all well and good, but how do we actually use our Barista actor in our application? To do that, we have to create a new instance of our Barista actor. You might be tempted to do it the usual way, by calling its constructor like so:

~~~
val barista = new Barista // will throw exception
~~~

This will not work! Akka will thank you with an ActorInitializationException. The thing is, in order for the whole actor thingie to work properly, your actors need to be managed by the ActorSystem and its components. Hence, you have to ask the actor system for a new instance of your actor:

~~~
import akka.actor.{ActorRef, Props}
val barista: ActorRef = system.actorOf(Props[Barista], "Barista")
~~~

The actorOf method defined on ActorSystem expects a Props instance, which provides a means of configuring newly created actors, and, optionally, a name for your actor instance. We are using the simplest form of creating such a Props instance, providing the apply method of the companion object with a type parameter. Akka will then create a new instance of the actor of the given type by calling its default constructor.

Be aware that the type of the object returned by actorOf is not Barista, but ActorRef. Actors never communicate with another directly and hence there are supposed to be no direct references to actor instances. Instead, actors or other components of your application aquire references to the actors they need to send messages to.

Thus, an ActorRef acts as some kind of proxy to the actual actor. This is convenient because an ActorRef can be serialized, allowing it to be a proxy for a remote actor on some other machine. For the component aquiring an ActorRef, the location of the actor – local in the same JVM or remote on some other machine – is completely transparent. We call this property location transparency.

Please note that ActorRef is not parameterized by type. Any ActorRef can be exchanged for another, allowing you to send arbitrary messages to any ActorRef. This is by design and, as already mentioned above, allows for easily modifying the topology of your actor system wihout having to make any changes to the senders.

### Sending messages

Now that we have created an instance of our Barista actor and got an ActorRef linked to it, we can send it a message. This is done by calling the ! method on the ActorRef:

~~~
barista ! CappuccinoRequest
barista ! EspressoRequest
println("I ordered a cappuccino and an espresso")
~~~

Calling the ! is a fire-and-forget operation: You tell the Barista that you want a cappuccino, but you don’t wait for their response. It’s the most common way in Akka for interacting with other actors. By calling this method, you tell Akka to enqueue your message in the recipient’s mailbox. As described above, this doesn’t block, and eventually the recipient actor will process your message.

Due to the asynchronous nature, the result of the above code is not deterministic. It might look like this:

~~~
I have to prepare a cappuccino!
I ordered a cappuccino and an espresso
Let's prepare an espresso.
~~~

Even though we first sent the two messages to the Barista actor’s mailbox, between the processing of the first and second message, our own output is printed to the console.

### Answering to messages

Sometimes, being able to tell others what to do just doesn’t cut it. You would like to be able to answer by in turn sending a message to the sender of a message you got – all asynchronously of course.

To enable you to do that and lots of other things that are of no concern to us right now, actors have a method called sender, which returns the ActorRef of the sender of the last message, i.e. the one you are currently processing.

But how does it know about that sender? The answer can be found in the signature of the ! method, which has a second, implicit parameter list:

~~~
def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit
~~~

When called from an actor, its ActorRef is passed on as the implicit sender argument.

Let’s change our Barista so that they immediately send a Bill to the sender of a CoffeeRequest before printing their usual output to the console:

~~~
case class Bill(cents: Int)
case object ClosingTime
class Barista extends Actor {
  def receive = {
    case CappuccinoRequest =>
      sender ! Bill(250)
      println("I have to prepare a cappuccino!")
    case EspressoRequest =>
      sender ! Bill(200)
      println("Let's prepare an espresso.")
    case ClosingTime => context.system.shutdown()
  }
}
~~~

While we are at it, we are introducing a new message, ClosingTime. The Barista reacts to it by shutting down the actor system, which they, like all actors, can access via their ActorContext.

Now, let’s introduce a second actor representing a customer:

~~~
case object CaffeineWithdrawalWarning
class Customer(caffeineSource: ActorRef) extends Actor {
  def receive = {
    case CaffeineWithdrawalWarning => caffeineSource ! EspressoRequest
    case Bill(cents) => println(s"I have to pay $cents cents, or else!")
  }
}
~~~

This actor is a real coffee junkie, so it needs to be able to order new coffee. We pass it an ActorRef in the constructor – for the Customer, this is simply its caffeineSource – it doesn’t know whether this ActorRef points to a Barista or something else. It knows that it can send CoffeeRequest messages to it, and that is all that matters to them.

Finally, we need to create these two actors and send the customer a CaffeineWithdrawalWarning to get things rolling:

~~~
val barista = system.actorOf(Props[Barista], "Barista")
val customer = system.actorOf(Props(classOf[Customer], barista), "Customer")
customer ! CaffeineWithdrawalWarning
barista ! ClosingTime
~~~

Here, for the Customer actor, we are using a different factory method for creating a Props instance: We pass in the type of the actor we want to have instantiated as well as the constructor arguments that actor takes. We need to do this because we want to pass the ActorRef of our Barista actor to the constructor of the Customer actor.

Sending the CaffeineWithdrawalWarning to the customer makes it send an EspressoRequest to the barista who will then send a Bill back to the customer. The output of this may look like this:

~~~
Let's prepare an espresso.
I have to pay 200 cents, or else!
~~~

First, while processing the EspressoRequest message, the Barista sends a message to the sender of that message, the Customer actor. However, this operation doesn’t block until the latter processes it. The Barista actor can continue processing the EspressoRequest immediately, and does this by printing to the console. Shortly after, the Customer starts to process the Bill message and in turn prints to the console.

### Asking questions

Sometimes, sending an actor a message and expecting a message in return at some later time isn’t an option – the most common place where this is the case is in components that need to interface with actors, but are not actors themselves. Living outside of the actor world, they cannot receive messages.

For situations such as these, there is Akka’s ask support, which provides some sort of bridge between actor-based and future-based concurrency. From the client perspective, it works like this:

~~~
import akka.pattern.ask
import akka.util.Timeout
import scala.concurrent.duration._
implicit val timeout = Timeout(2.second)
implicit val ec = system.dispatcher
val f: Future[Any] = barista2 ? CappuccinoRequest
f.onSuccess {
  case Bill(cents) => println(s"Will pay $cents cents for a cappuccino")
}
~~~

First, you need to import support for the ask syntax and create an implicit timeout for the Future returned by the ? method. Also, the Future needs an ExecutionContext. Here, we simply use the default dispatcher of our ActorSystem, which is conveniently also an ExecutionContext.

As you can see, the returned future is untyped – it’s a Future[Any]. This shouldn’t come as a surprise, since it’s really a received message from an actor, and those are untyped, too.

For the actor that is being asked, this is actually the same as sending some message to the sender of a processed message. This is why asking our Barista works out of the box without having to change anything in our Barista actor.

Once the actor being asked sends a message to the sender, the Promise belonging to the returned Future is completed.

Generally, telling is preferable to asking, because it’s more resource-sparing. Akka is not for polite people! However, there are situations where you really need to ask, and then it’s perfectly fine to do so.

### Stateful actors

Each actor may maintain an internal state, but that’s not strictly necessary. Sometimes, a large part of the overall application state consists of the information carried by the immutable messages passed between actors.

An actor only ever processes one message at a time. While doing so, it may modify its internal state. This means that there is some kind of mutable state in an actor, but since each message is processed in isolation, there is no way the internal state of our actor can get messed up due to concurrency problems.

To illustrate, let’s turn our stateless Barista into an actor carrying state, by simply counting the number of orders:

~~~
class Barista extends Actor {
  var cappuccinoCount = 0
  var espressoCount = 0
  def receive = {
    case CappuccinoRequest =>
      sender ! Bill(250)
      cappuccinoCount += 1
      println(s"I have to prepare cappuccino #$cappuccinoCount")
    case EspressoRequest =>
      sender ! Bill(200)
      espressoCount += 1
      println(s"Let's prepare espresso #$espressoCount.")
    case ClosingTime => context.system.shutdown()
  }
}
~~~

We introduced two vars, cappuccinoCount and espressoCount that are incremented with each respective order. This is actually the first time in this series that we have used a var. While to be avoided in functional programming, they are really the only way to allow your actors to carry state. Since each message is processed in isolation, our above code is similar to using AtomicInteger values in a non-actor environment.

Conclusion
-------------------------------------------------------

And here ends our introduction to the actor programming model for concurrency and how to work within this paradigm using Akka. While we have really only scratched the surface and have ignored some important concepts of Akka, I hope to have given enough of an insight into this approach to concurrency to give you a basic understanding and get you interested in learning more.

In the coming articles, I will elaborate our little example, adding some meaningful behaviour to it while introducing more of the ideas behind Akka actors, among them the question of how errors are handled in an actor system.

P.S. Please note that starting with this article I have switched to a biweekly schedule for the remaining parts of this series.

