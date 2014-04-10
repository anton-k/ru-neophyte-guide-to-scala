Глава 15: Обработка ошибок в системе акторов
=====================================================================

В предыдущей главе мы познакомились с моделью акторов -- вторым основопологающим понятием
параллельных вычислений. Она дополняет модельоснованную на компануемых асинхронных 
вычислениях (`Future` и `Promise`). Мы узнаели как акторы определяются, как отправлять
актору сообщения, как сообщения  обрабатываются, как актор может обновлять изменяемое
состояние при обработке сообщений или отправлять ответное сообщение отправитею сообщения.

Надеюсь этого материала было вполне достаточно для того, чтобы заинтересовать вас
моделью акторов. Но мы не конснулись ряда ключевых понятий, необходимых для построения
более менеее серьёзных приложений, основанных на акторах. 

Модель акторов предназначена для построения отказоустойчивых приложений. 
в этой статье мы посмотрим как реагировать на ошибки в приложениях,
основанных на акторах. Обработка ошибок в системах акторов сильно отличается
от традиционного подхода, принятого при построении серверов.

Способ обработки ошибок тесно связан с понятиями Akka и некоторыми элементами,
из которых состоит система акторов в Akka. Поэтому в данной статье мы продолжим 
изучение реализации модели акторов в Akka.

Иерархии акторов
-----------------------------------------------------------------------

Перед тем как обратиться к тому, что происходит когда возникает ошибка, 
необходимо усвоить одно из ключевых понятий в модели акторов. Это понятие
лежит в основе построения отказоустойчивых приложений. Акторы образуют
иерархии.

Но что это значит? Во первых, это означает что у каждого актора есть актор-родитель, 
и каждый актор может создавать дочерние акторы. Родительские акторы наблюдают за своими детьми,
точно так же как и в настоящей жизни. Они заботятся о них, помогают им подняться на ноги, 
если они споткнутся. Вскоре мы узнаем как это происходит.

### Актор-охранник

В прошлой статье мы успели определить два актора -- для бармена и посетителя. 
Я не буду повторять определения. Они очень простые, давайте сосредоточимся
на том, как создавались значения для акторов:

~~~
import akka.actor.ActorSystem

val system = ActorSystem("Coffeehouse")
val barista = system.actorOf(Props[Barista], "Barista")
val customer = system.actorOf(Props(classOf[Customer], barista), "Customer")
~~~

Мы создаём два актора вызовом метода `actorOf`, что определён на значении типа `ActorSystem`.

Кто является родителем для этих двух акторов? Система акторов? Не совсем верно, но близко. 
Система акторов сама не является актором, но в ней определён так называемы *актор-охранник*
(guardian actor), который является родителем для всех корневых акторов, то есть тех, которыx 
мы создаём вызовом метода `actorOf` на `ActorSystem`.

В нашей системе должно быть совсем немного таких акторов. Оправдано иметь лишь несколько
корневых акторов, каждый из которых делегирует большую часть своей работы дочерним акторам.

### Пути для акторов

Иерархическая структура системы акторов становится очевидной, если мы взглянем на пути
для созданных акторов. Пути -- это по сути URL для акторов, по которым мы можем ссылаться
на акторы. Мы можем получить путь к актору вызовом метода `path` на его ссылке `ActorRef`:

~~~
barista.path // => akka.actor.ActorPath = akka://Coffeehouse/user/Barista
customer.path // => akka.actor.ActorPath = akka://Coffeehouse/user/Customer
~~~

За протоколом `akka` следует имя нашей системы акторов, потом имя пользовательского 
актора-охранника, и наконец имя, которое мы дали актору при создании, в вызове `actorOf`.
В случае распределённых систем для акторов, работающих на удалённых машинах
мы увидим имя хоста и порт. 

Пути к акторам могут быть использованы для поиска акторов. к примеру вместо того, чтобы
передать `Customer` ссылку на актор бармена, мы могли бы найти этот актор вызовом метода
`actorSelection` на `ActorContext`, передав путь в качестве аргумента:

~~~
context.actorSelection("../Barista")
~~~

Однако лучше передать зависимость от актора в конструктор посетителя, так как
мы делали это раньше. Зависимость от зашитых в код путей к акторам может
привести к возникновению багов, и такой код гораздо труднее изменять.

### Пример иерархии

Давайте разовьёв наш пример с кафетерием, для того чтобы разбераться как родительские 
акторы могут следить за дочерними и как это может помочь нам в повышении отказоустойчивости 
приложения. Определим дочерний актор для бармена, теперь бармен сможет передать часть
работы своему дочернему актору.

Нам следовало бы определить сразу несколько дочерних акторов под разные задачи бармена,
но мы не будем распыляться и немного упростим нашу модель.


Предположим что в баре есть кассовый аппарат (`Register`), который  создаёт чеки и 
обновляет счётчик общих продаж за день. Вот пробная версия такого актора:

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

Он содержит неизменяемый ассоциативный массив для цен по каждому товару и переменную,
которая обозначает общий доход. Как только он получает сообщение `Transaction`,
он обновляет счётчик дохода и возвращает чек отправителю.

Актор `Register` будет дочерним актором бармена, поэтому мы создадим его
не из системы акторов, а из актора `Barista`. Исходная версия нашего родительского
актора выглядит так:

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

Сначала мы определили тип сообщений для нашего бармена. Чашка эсспрессо `EspressoCup`
может быть в одном из состояний, этот набор зафиксирован с помощью ключевого слова `sealed`. 

Но посмотрим, что происходит в классе `Barista`. Мы импортировали `dispatcher`, `ask` и `pipe`,
объявили неявную переменную `timeout`, потому что мы пользуемся синхронным обменом
сообщений с помощью `?`. Как только мы получаем `EspressoRequest`, мы спрашиваем у нашего
актора чек, отправив сообщение `Transaction`. Затем мы наливаем эспрессо в чашку, присоединяем
чек и перенаправляем отправителю заказа. Он получит пару `(EspressoCup, Receipt)`. 
Мы рассмотрели довольно типичный сценарий для приложений основанных на акторах. 
В акторе мы распределили обязанности между дочерними акторами и затем собрали полученне
данные.

Также обратите внимание на то, как мы создали дочерний актор вызовом `actorOf` на `ActorContext`
вместо `ActorSystem`. После этого созданный актор становится дочерним для того актора, который
вызвал метод `actorOf` на своём контексте, а не корневым актором, для которого родителем является
актор-охранник.

Наконец определим актор для посетителя `Customer`. Он является корневым также как и актор бармена:

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

Это определение не так интересно для рассматриваемых в данной главе вопросов. 
Стоит обратить внимание на лишь на то, как мы воспользвались трэйтом `ActorLogging`.
Он позволяет нам писать сообщения в лог вместо консоли.

Теперь если мы создадим систему акторов с барменом и двумя посетителями, мы можем
напоить двух наших кофеманов, чашечкой чёрного золота: 

~~~
import Customer._

val system = ActorSystem("Coffeehouse")

val barista = system.actorOf(Props[Barista], "Barista")
val customerJohnny = system.actorOf(Props(classOf[Customer], barista), "Johnny")
val customerAlina = system.actorOf(Props(classOf[Customer], barista), "Alina")

customerJohnny ! CaffeineWithdrawalWarning
customerAlina ! CaffeineWithdrawalWarning
~~~

После запуска этого примера Вы увидете два сообщения в логе от
двух радостных посетителей.


Падать или не падать?
--------------------------------------------------------------

Но в этой статье мы заинтересованы совсем не в радостных посетителях, нам интересно узнать,
что происходит, когда что-то идёт не так.

Наш кассовый аппарат не так надёжен как нам бы хотелось. Он может зажевать бумагу.
Давайте добавим исключение для этого случая, в объект-компаньон для `Register`:

~~~
class PaperJamException(msg: String) extends Exception(msg)
~~~

Теперь давайте изменим метод `createReceipt` в нашем акторе `Register`:

~~~
def createReceipt(price: Int): Receipt = {
  import util.Random
  if (Random.nextBoolean())
    throw new PaperJamException("OMG, not again!")
  Receipt(price)
}
~~~

Теперь при обработке сообщения `Transaction`, наш актор `Register` будет выдавать
исключение в половине случаев.

Как это скажется на нашей системе акторов? К счастью Akka -- очень надёжна, исключения
не приведут к падению приложения, вместо этого дочерний актор будет оповещён
о том, что с дочерним актором что-то случилось. И в этом случае у дочернего актора
может быть несколько вариантов разрешения ситуации.

### Стратегии наблюдателя


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

