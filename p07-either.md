Глава 7: Тип Either
================================================

В предыдущей главе мы узнали как обрабатываются исключения в функциональном стиле.
Мы узнали о тип `Try`. Он появился в Scala с версии 2.10. Также я упомнул и о типе `Either`.
В этой главе мы остановимся на нём по-подробнее. Мы узнаем как и где он используется и о некоторых
неприятных особенностях типа `Either`. 

На данный момент в типе `Either` есть несколько серьёзных недостатков, о которых стоит знать. 
Они настолько серьёзны, что необходимость использования `Either` может вызывать сомнения. 
А нужен ли он нам?

Во первых, до появления `Try` для обработки исключений использовался тип `Either` и не
все разработчики перешли на новый способ. Поэтому нам стоит разобраться в тонкостях и этого типа.

Кроме того, `Try` не равносилен `Either`. `Try` используется только для обработки исключений
в функциональном стиле. На прктике `Try` и `Either` дополняют друг друга. И несмотря на все
недостатки `Either`, есть проблемы, с которыми он справляется очень хорошо.

Семантика
-----------------------------------------

Как `Try` и `Option` тип `Either` является контейнером. Только он принимает два параметра, а не один: 
значение типа `Either[A, B]` может содержать значение типа `A` или значение типа `B`. В этом отличие от `Tuple[A, B]`,
который одновременно содержит два значения,  типов `A` и `B`.

От `Either` наследуют всего два класса: `Left` и `Right`. Если значение `Either[A, B]` содержит 
значение типа `A`,  тогда `Either` содержит `Left`, иначе оно содержит значение типа `B` 
обёрнутое в класс `Right`.

В семантике типа ничто не указывает на то, что один или другой тип представляет ошибку
или успешное выполнение. На самом деле, `Either` обозначает тип общего назначения, 
представляющий результат, в котором для значений есть две возможные альтернативы. 
Но несмотря на это, чаще всего он используется для обработки исключений, и по соглашению
`Left` отвечает за ошибки/исключения, а `Right` -- за успешно вычисленное значение.

Создание значения типа `Either`
------------------------------------------

Значения типа `Either` создаются очень просто. И `Left` и `Right` являются `case`-классами. 
Так если мы хотим реализовать непробиваемую систему интернет-цензуры, мы можем сделать это так:

~~~
import scala.io.Source
import java.net.URL

def getContent(url: URL): Either[String, Source] =
  if (url.getHost.contains("google"))
    Left("Requested URL is blocked for the good of the people!")
  else
    Right(Source.fromURL(url))
~~~

Теперь, если мы вызовим `getContent(new URL("http://danielwestheide.com"))`, то мы получим
`scala.io.Source` обёрнутый в `Right`. Если мы попробуем обратиться по `new URL("https://plus.google.com")`,
результат будет содержать `Left` со строкой.

Использование `Either`
----------------------------------------------

Некоторые совсем простые вещи работают в `Either` точно так же как и в `Try`. У нас есть методы `isLeft`
и `isRight`. Также мы можем выполнять сопоставление с образцом, этот способ работы с типом `Either`
наиболее удобен:

~~~
getContent(new URL("http://google.com")) match {
  case Left(msg) => println(msg)
  case Right(source) => source.getLines.foreach(println)
}
~~~

### Проекции

Мы не можем работать с `Either` как с коллекциями, также как и в `Option` или `Try`. 
Это решение вызвано тем, что `Either` не должно отдавать предпочтение одной или другой альтернативе.

В `Try` мы концентрируемся на типе результата, успешно вычисленного значения, для него определены
`map`, `flatMap` и другие методы. Все они предполагают, что `Try` содержит `Success`, и если это
не так, они ничего не делают, передавая далее `Failure`. 

Поскольку в `Either` альтернативы равнозначны, мы сначаал должны определиться с какой веткой 
мы хотим работать, вызовом методов `left` или `right` на значении типа `Either`. После этого
мы получим одну из проекций `LeftProjection` или `RightProjection`, которые концентрируются на 
левой и правой альтернативе соответственно. 

#### Преобразование

После того как у нас есть проекция, мы можем преобразовать её элемент:

~~~
val content: Either[String, Iterator[String]] =
  getContent(new URL("http://danielwestheide.com")).right.map(_.getLines())
// content содержит Right со строчками из Source, который был получен с помощью getContent

val moreContent: Either[String, Iterator[String]] =
  getContent(new URL("http://google.com")).right.map(_.getLines)
// moreContent содержит Left, полученный из getContent
~~~

Что бы не содержало значение `Either[String, Source]` в этом примере, `Left` или `Right`, оно будет
преобразовано в `Either[String, Iterator[String]]`. Если оно содержит `Right`, то значение будет 
преобразовано, если оно содержит `Left`, значение останется без изменений.

То же самое мы можем выполнить и для `LeftProjection`:

~~~
val content: Either[Iterator[String], Source] =
  getContent(new URL("http://danielwestheide.com")).left.map(Iterator(_))
// content содержит Right с Source, в том виде, в котором он был получен из getContent

val moreContent: Either[Iterator[String], Source] =
  getContent(new URL("http://google.com")).left.map(Iterator(_))
// moreContent содержит Left с msg, полученым из getContent в Iterator'е
~~~

Теперь, если `Either` содержит `Left`, результат будет преобразован, а в случае
`Right` оставден без изменений. И в том и вдругом случае результат значения будет
`Either[Iterator[String], Source]`

Обратите внимание на то, что метод `map` определён на проекциях, а не на самом типе `Either`,
но он возвращает значение типа `Either`, а не проекции. В этом `Either` уходит от привычной
аналогии с коллекциями. Так происходит потому, что в `Either` альтернативы должны оставаться
равнозначными. Но Вы уже наверное догадываетесь о том, какие проблемы могут стоять за этим
решением. к тому же, если мы хотим вызвать несколько методов `map`, `flatMap` и других методов, 
нам придётся каждый раз указывать какую проекцию мы хотим использовать. 

#### Метод `flatMap`

Для проекций также определён метод `flatMap`, который позволяет избежать проблемы вложенных структур
при вызове `map`. 

Прошу потерпеть, но сейчас мы рассмотрим очень надуманный пример. Допустим нам хочется
узнать среднее число строк для двух моих статей. Нам ведь давно хотелось узнать об этом, не так ли?
Мы можем решить эту сложнейшую задачу, примерно так:


I’m putting very high requirements on your suspension of disbelief now, coming up with a completely contrived example. Let’s say we want to calculate the average number of lines of two of my articles. You’ve always wanted to do that, right? Here’s how we could solve this challenging problem:

~~~
val part5 = new URL("http://t.co/UR1aalX4")
val part6 = new URL("http://t.co/6wlKwTmu")
val content = getContent(part5).right.map(a =>
  getContent(part6).right.map(b =>
    (a.getLines().size + b.getLines().size) / 2))
~~~

В результате мы получим значение типа `Either[String, Either[String, Int]]`, что содержит вложенный `Either`
в правой альтернативе. Мы можем избавиться от этой вложенности вызовом метода `joinRight`, также
определён и метод `joinLeft`. 

Однако мы можем не создавать вложенную структуру изначально. Если мы вызовем `flatMap` на `RightProjection`, то
мы полуим более приятный результат. Это приведёт к тому что, мы избавимся от `Right` во внутреннем `Either`. 

~~~
val content = getContent(part5).right.flatMap(a =>
  getContent(part6).right.map(b =>
    (a.getLines().size + b.getLines().size) / 2))
~~~

Теперь переменная content имеет тип `Either[String, Int]` и с ней гораздо приятнее работать,
например, при сопоставлении с образцом. 

#### For-генераторы


By now, you have probably learned to love working with for comprehensions in a consistent way on various different data types. You can do that, too, with projections of Either, but the sad truth is that it’s not quite as nice, and there are things that you won’t be able to do without resorting to ugly workarounds, out of the box.

Let’s rewrite our flatMap example, making use of for comprehensions instead:

~~~
def averageLineCount(url1: URL, url2: URL): Either[String, Int] =
  for {
    source1 <- getContent(url1).right
    source2 <- getContent(url2).right
  } yield (source1.getLines().size + source2.getLines().size) / 2
~~~

This is not too bad. Note that we have to call right on each Either we use in our generators, of course.

Now, let’s try to refactor this for comprehension – since the yield expression is a little too involved, we want to extract some parts of it into value definitions inside our for comprehension:

~~~
def averageLineCountWontCompile(url1: URL, url2: URL): Either[String, Int] =
  for {
    source1 <- getContent(url1).right
    source2 <- getContent(url2).right
    lines1 = source1.getLines().size
    lines2 = source2.getLines().size
  } yield (lines1 + lines2) / 2
~~~

This won’t compile! The reason will become clearer if we examine what this for comprehension corresponds to, if you take away the sugar. It translates to something that is similar to the following, albeit much less readable:

~~~
def averageLineCountDesugaredWontCompile(url1: URL, url2: URL): Either[String, Int] =
  getContent(url1).right.flatMap { source1 =>
    getContent(url2).right.map { source2 =>
      val lines1 = source1.getLines().size
      val lines2 = source2.getLines().size
      (lines1, lines2)
    }.map { case (x, y) => x + y / 2 }
  }
~~~

The problem is that by including a value definition in our for comprehension, a new call to map is introduced automatically – on the result of the previous call to map, which has returned an Either, not a RightProjection. As you know, Either doesn’t define a map method, making the compiler a little bit grumpy.

This is where Either shows us its ugly trollface. In this example, the value definitions are not strictly necessary. If they are, you can work around this problem, replacing any value definitions by generators, like this:

~~~
def averageLineCount(url1: URL, url2: URL): Either[String, Int] =
  for {
    source1 <- getContent(url1).right
    source2 <- getContent(url2).right
    lines1 <- Right(source1.getLines().size).right
    lines2 <- Right(source2.getLines().size).right
  } yield (lines1 + lines2) / 2
~~~

It’s important to be aware of these design flaws. They don’t make Either unusable, but can lead to serious headaches if you don’t have a clue what’s going on.

#### Other methods

The projection types have some other useful methods:

You can convert your Either instance to an Option by calling toOption on one of its projections. For example, if you have an e of type Either[A, B], e.right.toOption will return an Option[B]. If your Either[A, B] instance is a Right, that Option[B] will be a Some. If it’s a Left, it will be None. The reverse behaviour can be achieved, of course, when calling toOption on the LeftProjection of your Either[A, B]. If you need a sequence of either one value or none, use toSeq instead.

Folding
--------------------------------

If you want to transform an Either value regardless of whether it is a Left or a Right, you can do so by means of the fold method that is defined on Either, expecting two transform functions with the same result type, the first one being called if the Either is a Left, the second one if it’s a Right.

To demonstrate this, let’s combine the two mapping operations we implemented on the LeftProjection and the RightProjection above:

~~~
val content: Iterator[String] =
  getContent(new URL("http://danielwestheide.com")).fold(Iterator(_), _.getLines())
val moreContent: Iterator[String] =
  getContent(new URL("http://google.com")).fold(Iterator(_), _.getLines())
~~~

In this example, we are transforming our Either[String, Source] into an Iterator[String], no matter if it’s a Left or a Right. You could just as well return a new Either again or execute side-effects and return Unit from your two functions. As such, calling fold provides a nice alternative to pattern matching.

When to use Either
---------------------------------


Now that you have seen how to work with Either values and what you have to take care of, let’s move on to some specific use cases.

### Error handling

You can use Either for exception handling very much like Try. Either has one advantage over Try: you can have more specific error types at compile time, while Try uses Throwable all the time. This means that Either can be a good choice for expected errors.

You’d have to implement a method like this, delegating to the very useful Exception object from the scala.util.control package:

~~~
import scala.util.control.Exception.catching
def handling[Ex <: Throwable, T](exType: Class[Ex])(block: => T): Either[Ex, T] =
  catching(exType).either(block).asInstanceOf[Either[Ex, T]]
~~~

The reason you might want to do that is because while the methods provided by scala.util.Exception allow you to catch only certain types of exceptions, the resulting compile-time error type is always Throwable.

With this method at hand, you can pass along expected exceptions in an Either:

~~~
import java.net.MalformedURLException
def parseURL(url: String): Either[MalformedURLException, URL] =
  handling(classOf[MalformedURLException])(new URL(url))
~~~

You will have other expected error conditions, and not all of them result in third-party code throwing an exception you need to handle, as in the example above. In these cases, there is really no need to throw an exception yourself, only to catch it and wrap it in a Left. Instead, simply define your own error type, preferably as a case class, and return a Left wrapping an instance of that error type in your expected error condition occurs.

Here is an example:

~~~
case class Customer(age: Int)
class Cigarettes
case class UnderAgeFailure(age: Int, required: Int)
def buyCigarettes(customer: Customer): Either[UnderAgeFailure, Cigarettes] =
  if (customer.age < 16) Left(UnderAgeFailure(customer.age, 16))
  else Right(new Cigarettes)
~~~

You should avoid using Either for wrapping unexpected exceptions. Try does that better, without all the flaws you have to deal with when working with Either.

### Processing collections

Generally, Either is a pretty good fit if you want to process a collection, where for some items in that collection, this might result in a condition that is problematic, but should not directly result in an exception, which would result in aborting the processing of the rest of the collection.

Let’s assume that for our industry-standard web censorship system, we are using some kind of black list:

~~~
type Citizen = String
case class BlackListedResource(url: URL, visitors: Set[Citizen])

val blacklist = List(
  BlackListedResource(new URL("https://google.com"), Set("John Doe", "Johanna Doe")),
  BlackListedResource(new URL("http://yahoo.com"), Set.empty),
  BlackListedResource(new URL("https://maps.google.com"), Set("John Doe")),
  BlackListedResource(new URL("http://plus.google.com"), Set.empty)
)
~~~

A BlackListedResource represents the URL of a black-listed web page plus the citizens who have tried to visit that page.

Now we want to process this black list, where our main purpose is to identify problematic citizens, i.e. those that have tried to visit blocked pages. At the same time, we want to identify suspicous web pages – if not a single citizen has tried to visit a black-listed page, we must assume that our subjects are bypassing our filter somehow, and we need to investigate that.

Here is how we can process our black list:

~~~
val checkedBlacklist: List[Either[URL, Set[Citizen]]] =
  blacklist.map(resource =>
    if (resource.visitors.isEmpty) Left(resource.url)
    else Right(resource.visitors))
~~~

We have created a sequence of Either values, with the Left instances representing suspicious URLs and the Right ones containing sets of problem citizens. This makes it almost a breeze to identify both our problem citizens and our suspicious web pages:

~~~
val suspiciousResources = checkedBlacklist.flatMap(_.left.toOption)
val problemCitizens = checkedBlacklist.flatMap(_.right.toOption).flatten.toSet
~~~

These more general use cases beyond exception handling are where Either really shines.

Summary
-------------------------------------------

You have learned how to make use of Either, what its pitfalls are, and when to put it to use in your code. It’s a type that is not without flaws, and whether you want to have to deal with them and incorporate it in your own code is ultimately up to you.

In practice, you will notice that, now that we have Try at our hands, there won’t be terribly many use cases for it. Nevertheless, it’s good to know about it, both for those situations where it will be your perfect tool and to understand pre-2.10 Scala code you come across where it’s used for error handling.








