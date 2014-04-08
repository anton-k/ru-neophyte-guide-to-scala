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

Модель акторов может избавить нас от всех упомянутых выше проблем, позволяя
нам писать высокоэффективный но в то же время очень ясный код. Она принуждает
нас к тому, чтобы код был параллельным с самого начала. Ведь добавить распараллеливание
позже на практике не представляется возможным. 

Основная идея в том, что приложение построено из многих легковесных процессов, 
называемых акторами. Каждый актор отвечает за одну очень маленькую задачу,
поэтому нам легко понять, что он делает. Более сложная логика возникает из
взаимодействия нескольких акторов, мы решаем задачи с помощью одних акторов,
в это время посылаем сообения совокупности других. 

### Система акторов

Акторы -- несамостоятельные существа, они не могут жить сами по себе. Каждый актор
в Akka создаётся *системой акторов* (actor system). Кроме создания акторов и поиска,
система акторов `ActorSystem` позволяет нам выполнять множество других операций, о которых
мы пока умолчим. 

Для того чтобы код примеров запускался сначала добавьте следующие зависимости в SBT-проект
с компилятором Scala 2.10:

~~~
resolvers += "Typesafe Releases" at "http://repo.typesafe.com/typesafe/releases"

libraryDependencies += "com.typesafe.akka" %% "akka-actor" % "2.2.3"
~~~

Теперь давайте создадим систему акторов `ActorSystem`. Она будет выступать в роли 
среды обитания для наших акторов:

~~~
import akka.actor.ActorSystem

object Barista extends App {
  val system = ActorSystem("Barista")
  system.shutdown()
}
~~~

Мы создали новое значение типа `ActorSystem` и дали ей имя `"Barista"` (или "бармен"" по англ.). Мы возвращаемся к кофейному примеру,
с которым мы уже встречались в статье про `Future`.

Кроме того мы добропорядочно завершили работу системы акторов, по окончанию работы приложения.

### Определение акторов

Наше приложение может состоять из нескольких десятков или нескольких миллионов акторов, Akka
с этим прекрасно справляется. Но постойте-постойте, несколько миллионов? Вас может удивить это
невероятно огромное число. Важно понимать, что между акторами и потоками нет соответствия один к одному.
Если бы это было так мыбы очень скоро исчерпали бы всю память. Поскольку акторы не блокируют 
потоки вычисления, один поток может выполнять несколько акторов, перключаясь между ними, в зависимости
от того к какому актору приходят сообщения. 

Для иллюстрации давайте создадим очень простой актор. Бармен может принимать заказы, 
но пока он ничего не будет делать, только выводить сообщения напечать:

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

Мы определили несколько типов сообщений, которые принимает наш актор. 
Обычно для передачи сообщений, которые содержат значения, между акторами используются `case`-классы. 
Если сообщение ничего не содержит мы будем использовать `case`-объекты, как в данном примере.

В любом случае очень важно, чтобы наши сообщения были бы неизменяемыми значениями,
иначе могут произойти очень плохие вещи.

Давайте присмотримся к нашему классу `Barista` для описания бармена. Он наследует
от трэйта `Actor`. Этот трэйт определяет метод `receive`, который возвращает значения
типа `Receive`. Этот тип это просто синоним для `PartialFunction[Any, Unit]`.

### Обработка сообщений

Так в чём же суть метода `receive`? Тип `PartialFunction[Any, Unit]` может показаться странным сразу по нескольким причинам.

По смыслу частично определённая функция, возвращаемая методом `receive` отвечает за обработку сообщений.
Если к нам приходит сообщение из любой другой части приложения, будь то актор или нет, Akka 
вызовет метод `receive`, передав сообщение в качестве аргумента.

#### Выполнение побочных эффектов

Во время обработки сообщения актор может делать всё что угодно, кроме возвращения значения.

> Что за!?

Как видно из типа метода `receive` наша частично определённая функция выполняет побочные эффекты.
Это может отпугнуть поборников функционального программирования, ведь мы привыкли отдавать
предпочтение чистым функциям. И это справедливо и для параллельных программ. Но наше состояние
заключено внутри акторов и каждое сообщение обрабатывается одно за другим независимо от других акторов.
Поэтому у нас не возникает необходимости в синхронизации процессов или семафорах. Это здорово,
когда для побочных эффектов выделено отдельное место и они находятся под контролем. 

#### Отсутствие типизации

Но... этп частично определённая функция не только выполняет побочные эффекты, она ещё и не типизирована
настолько насколько это возможно в Scala. Мы можем передавать сообщения любого типа. 
Зачем нам это нужно, если у нас под рукой есть такая мощная система типов?

Это связано с некоторыми очень важными архитектурными решениями в Akka. Благодаря
этому мы можем перенаправлять сообщения другим акторам, управлять вычислительной нагрузкой
или создавать прокси акторы, в тайне от отправителя сообщений и так далее.

Иногда это и вправду приводит к весьма неприятным ошибкам, которые компилятор
не способен распознать. Для того чтобы сделать передачу сообщений более безопасной,
в ущерб некоторым возможностям нашей системы, в Akka мы можем воспользоваться экспериментальным
типом `Channel`.

#### Асинхронные и неблокирующие

Ранее говорилось о том, что в Akka наши акторы рано или поздно обработают
отправленные им сообщения. Важно опнимать, что отправление сообщения и
 приём происходят асинхронно, без блокирования потока вычислений. Отправитель
 не будет заблокирован во время обработки сообщения. Вместо этого он продолжит
 выполнять свою работу. Возможно отправитель хочет получить ответ от нашего актора,
 но вполне возможно, что ответ ему совсем не важен.

На самом деле, когда мы отправляем сообщение, оно доставляется в почтовый ящик актора,
который представляет собой очередь. Добавление сообщения в почтовый ящик не блокирует 
вычисления, то есть отправитель не ждёт пока сообщение будет добавлено в очередь адресата. 

Обработчик событий заметит появление нового сообщения, опять же асинхронно. 
Если актор не занимается обработкой другого сообщения, для него выделяется отдельный 
поток, доступный в контексте вычислений. Как только актор закончит обработку предыдущего сообщения, 
обработчик отправит ему следующее сообщение из почтового ящика.

Актор блокирует выделенный ему поток вычисляений до тех пор, пока не закончится обработка сообщения.
Хотя это не скажется на отправителе, это всё же означает, что медленные операции могут затормозить
всё приложение, поскольку для остальных акторов останется меньше потоков. 

Поэтому необходимо как можно быстрее проводить обработку сообщений и по возможности
стараться избегать вызовов блокирующих операций. 

Конечно мы не можем избежать возникновения таких ситуаций. Большинство драйверов для баз данных
по-прежнему блокируют потоки вычисления, и нам скорее всего захочется пользоваться базами данных
и в нашем приложении основанном на акторах. Эта проблема имеет решение, но мы пока не коснёмся
его. Оно выходит за рамки ознакомительного материала.

### Создание акторов

Пока мы научились определять акторы, но как они создаются? Как мы создадим актор для нашего 
бармена? Для этого нам нужно создать новое значение для актора `Barista`. Возможно Вам захочется
сделать это так:

~~~
val barista = new Barista // приведёт к исключению
~~~

Но это не сработает! Akka ответит нам исключением `ActorInitializationException`. 
Суть в том, что для успешной работы акторов, они должны управляться системой `ActorSystem`
и её сервисами. Поэтому нам нужно создать новый актор через систему акторов:

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

