Глава 8: Добро пожаловать в Будущее
====================================================

Возможно Вы что-то слышали о том как устроена параллелизация программ в Scala,
быть может именно поэтому Вы и заинтересовались Scala. Средства, предоставляемые
Scala, очень сильно упрощают задачу построения параллельных приложений, в сравнении
с теми низкоуровневыми интерфейсами, что доступны в большинстве других языков. 

Параллельные вычисления в Scala основаны на двух столбах, один из них -- `Future`,
другой -- `Actor`. О первом мы и поговорим в этой статье. Я объясню чем хорош 
этот тип и как с ним работать в функциональном стиле. 

Убедитесь в том, что у вас установлен компилятор Scala версии не ниже 2.9.3,
иначе примеры не будут работать. Этот тип появился в версии 2.10 и был портирован 
для 2.9.3, изначально в немного другом виде он входил в состав библиотеки 
для параллельных вычислений Akka.

Когда последовательный код плох?
------------------------------------------------------

Предположим, нам нужно приготовить капучино. Для этого просто
выполним последовательность действий одно за другим:

1. Помолим кофейные зёрна
2. Вскипятим воду
3. Сварим эспрессо, смешав молотые зёрна с кипятком
4. Взобьём молоко
5. Добавим молоко в эспрессо и капучино готово

Если перевести в Scala, мы получим следующее:

~~~
import scala.util.Try

// Определим осмысленные синонимы: 
type CoffeeBeans = String
type GroundCoffee = String
case class Water(temperature: Int)
type Milk = String
type FrothedMilk = String
type Espresso = String
type Cappuccino = String

// Методы-заглушки для отдельных шагов алгоритма:
def grind(beans: CoffeeBeans): GroundCoffee = s"ground coffee of $beans"
def heatWater(water: Water): Water = water.copy(temperature = 85)
def frothMilk(milk: Milk): FrothedMilk = s"frothed $milk"
def brew(coffee: GroundCoffee, heatedWater: Water): Espresso = "espresso"
def combine(espresso: Espresso, frothedMilk: FrothedMilk): Cappuccino = "cappuccino"

// Исключения, на случай если что-то пойдёт не так
// (они понадобяться нам позже):
case class GrindingException(msg: String) extends Exception(msg)
case class FrothingException(msg: String) extends Exception(msg)
case class WaterBoilingException(msg: String) extends Exception(msg)
case class BrewingException(msg: String) extends Exception(msg)

// последовательно выполним алгоритм:
def prepareCappuccino(): Try[Cappuccino] = for {
  ground <- Try(grind("arabica beans"))
  water <- Try(heatWater(Water(25)))
  espresso <- Try(brew(ground, water))
  foam <- Try(frothMilk("milk"))
} yield combine(espresso, foam)
~~~

У такого подхода есть несколько преимуществ: у нас есть очень простой пошаговый рецепт, 
мы знаем что нам делать и мы не собьёмся, поскольку у нас нет необходимости в переключении
контекста.  

Но есть и свои минусы. Нам часто приходиться ждать впустую. Мы не можем ничего делать пока 
мы ждём приготовления зеёрен. Ведь мы собираемся перейти к следующему шагу только тогда,
когда закончиться предыдущий.

Очевидно, что мы теряем много времени. Скорее всего нам захочется начать сразу несколько 
действий из этого списка одновременно. Как только зёрна и вода будут готовы, мы начнём делать
эспрессо, по ходу взбивая молоко. 

Точно так же дела обстоят и в программировании. Множество потоков ждут ответа от веб-сервера.
Мы не хотим, чтобы все они были заблокированы выполнением какого-нибудь запроса к базе 
или вызовом HTTP-сервиса. В этом случае мы можем воспользоваться асинхронными вычислениями
и неблокирующим IO, так чтобы при вызове одним из запросов базы данных, веб-сервер мог
заниматься другими запросами, а не простаивать в пустую в ожидании ответа от базы.

> “Я слышал, Вам нравятся функции обратного вызова, и я поместил одну функцию обратного вызова в другоую!”

Конечно, Вы наслышаны обо всём этом через шумиху вокруг Node.js. Подход Node.js заключается в использовании
функций обратного вызова (callback). К сожалению, при таком подходе код легко превращается в кашу из 
функций обратного вызова, в других функциях, которые сами в других функциях, которые .... в других функциях
обратного вызова. Такой код крайне трудно читать и  отлаживать.

В Scala мы тоже можем пользоваться функциями обратного вызова через `Future`, но нам они не понадобятся.
Есть лучшие альтернативы.

> “Я знаю о типе `Future`. Он совершенно бесполезен!”

Вам, возможно, встречалась реализация `Future` в Java. Всё что мы можем делать с `Future` в Java
так это проверить завершилось оно или нет, или блокировать вычисления до завершения. Этот тип
практически бесполезен и работать с ним не так сладко. 

Но Вы будете приятно удивлены тем, как устроен тип `Future` в Scala. Так приступим!

Семантика Future
----------------------------------------

Тип `Future[T]`,  определённый в scala.concurrent package -- это тип контэйнер, представляющий вычисление, которое 
когда-нибудь закончится и вернёт значение типа `T`. Вычисление может закончиться с ошибкой или не буть вычисленным
в поставленные временные рамки. Если что-то пойдёт не так, то результат будет содержать исключение.

`Future` это контейнер для однократной записи, когда вычисление будет завершено, значение этого типа неизменяемое. 
Также `Future` предоставляет методы, позволяющие считать вычисляемое значение. Запись значения осуществляется
с помощью типа `Promise`. Эти понятия чётко разделены в интерфейсе. Данная статья посвящена `Future`, а о `Promise`
мы поговорим в следующей. 

Работа с Future
---------------------------------------

Существует несколько способов использования Future, мы посмотрим на них на нашем кофейном примере. 
Во первых, нам нужно переписать все функции для отдельных шагов так, чтобы они
сразу возвращали результат в виде `Future`, а не блокировали бы вычисления.

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

Что же происходит в примере? Поясним моменты связанные с `Future`.

Во первых, в объекте-компаньоне для `Future` определён метод `apply`,
он принимает два аргумента:

~~~
object Future {
  def apply[T](body: => T)(implicit execctx: ExecutionContext): Future[T]
}
~~~

В параметр `body` по имени передаётся вычисление, которое будет выполняться асинхронно. 
Второй параметр -- контекст вычисления, заключённый в отдельный список параметров, является имплицитным (implicit), 
это означает, что мы можем не передавать его явно, если значение с таким типом определено
в той же области видимости переменных. В случае `Future` для этого мы импортируем глобальный
контекст вычисления.

Значение типа `ExecutionContext` это то, что нужно для выполнения асинхронных вычислений, мы
можем представить что это пул потоков. Поскольку этот параметр передаётся неявно, в нашем методе `apply`
остаётся лишь один аргумент. Для передачи одного параметра в Scala мы можем воспользоваться как круглыми
так и фигурными скобками. Очень часто для вызова `Future` используются именно фигурные скобки,
словно это не обычный вызов, а применение некоторой встроенной фуннкции языка. Неявная передача 
`ExecutionContext` происходит почти во всех методах интерфейса для `Future`.

В нашем кофейном примере нам не приходится ничего вычислять, поэтому, мы просто 
вставили задержку потока вычисления с некоторым произвольным временем.  
Также мы выводим сообщения на печать до и после задержки, для того чтобы увидеть
неопредлённость, присущую параллельным вычислениям в нашем примере. 

Вычисление возвращаемого значения в `Future` начнётся в некотором потоке из 
`ExecutionContext` в неопределённое время после создания значения типа `Future`. 


Функции обратного вызова
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


