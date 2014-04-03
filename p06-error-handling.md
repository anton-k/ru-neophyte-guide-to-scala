Глава 6: Обработка исключений с помощью Try
===========================================

При знакомстве с новым языком мы можем писать простые программы, забывая о возможности
появления исключительных ситуаций. Но мы не можем игнорировать их, если мы собираемся
писать что-либо серьёзное. Часто по той или иной причине возможности языка, 
связанные с обработкой исключений, недооцениваются.

как оказывается в Scala предусмотрены весьма элегантные средства для раюоты с ошибками.
В этой статье мы познакомимся с типом `Try`, основой для обработки исключений в Scala.
Эти возможности стали доступны начиная с версии 2.10, они также портированы в 2.9.3.
Так что убедитесь в том, что Вы пользуетесь компилятором Scala версии не ниже чем 2.9.3.

Создание и обработка исключений
--------------------------------------------------

Перед тем как перейти к идиоматичному способу обработки исключений, давайте посмотрим
на способ, знакомый нам по другим языкам, таким как Java или Ruby. В Scala мы можем создавать
исключения точно так же:

~~~
case class Customer(age: Int)

class Cigarettes

case class UnderAgeException(message: String) extends Exception(message)

def buyCigarettes(customer: Customer): Cigarettes =
  if (customer.age < 16)
    throw UnderAgeException(s"Customer must be older than 16 but was ${customer.age}")
  else new Cigarettes
~~~

Созданные исключения могут быть обработаны так же как и в Java, только в Scala
мы воспользуемся сопоставлением с образцом для того, чтобы определить какие исключения
мы хотим обработать:

~~~
val youngCustomer = Customer(15)
try {
  buyCigarettes(youngCustomer)
  "Yo, here are your cancer sticks! Happy smokin'!"
} catch {
    case UnderAgeException(msg) => msg
}
~~~

Обработка исключений в функциональном стиле
--------------------------------------------------

Мы можем очень быстро засорить наше приложение таким способом обработки исключений. Но он совсем не
вяжется с функциональным программированием. Также это решение плохо подходит для параллельных приложений.
К примеру, если исключение случится в акторе, который выполняется в другом потоке, мы не сможем обработать
это исключение, вместо этого нам бы хотелось получить сообщение, обозначающее исключительную ситуацию.

Поэтому в Scala, предпочтительным способом обозначения исключений является возвращение специального
значения из функции. 

Но не волнуйтесь, мы вовсе не собираемся возвращаться к кодам ошибок, принятым в C, проверка которых 
обусловлена лишь соглашениями. В Scala мы воспользуемся спеуиальным типом, который говорит о том, что 
вычисление может привести к ошибке.

В этой главе мы ограничимся рассмотрением типа `Try`, которы был включён в Scala с версии 2.10 и
позднее портирован в 2.9.3. Также есть очень похожий тип `Either`, на практике возникают ситуации, когда
нам нужен именно этот тип, но он более общий.

### Семантика Try

Смысл типа `Try` лучше всего объяснить, сравнив его с типом `Option`, о котором шла речь в предыдущей статье.

В то время как `Option[A]` предсьавляет собой контейнер, который может содержать одно значение типа `A`
или быть пустым, тип `Try[A]` описывает вычисление, которое может вернуть значение типа `A`, если оно завершиться
успешно или некоторый объект из класса `Throwable`, если что-то пойдёт не так. Значения типа `Try[A]` могут
быть лего прееданы между разными одновременно выполняющимися частями приложения. 

Существует два варианта для типа `Try`. Если вычисление окончилось успешно `Try[A]` содержит 
объект типа `Success[A]`, это просто значение типа `A` обёрнутое в конструктор `Success`. 
Если вычисление завершилось ошибкой, значение содержит объект из класса `Failure[A]`
с исключением типа `Throwable`.  

Если нам известно, что вычисление может закончиться с ошибкой, то мы можем просто
использовать `Try[A]` в качестве типа возвращаемого значения. Благодаря чему пользователь
нашей функции по сигнатуре типу будет знать о том, что что-то может пойти не так. 

Предположим, что нам нужно написать простой загрузчик веб-страниц. Пользователь 
может набрать URL страницы, которую он хочет просмотреть. Где-то в приложении нам
нужно разобрать URL из строки и преобразовать его в `java.net.URL`:

~~~
import scala.util.Try
import java.net.URL
def parseURL(url: String): Try[URL] = Try(new URL(url))
~~~

Как видно из определения мы возвращаем результат типа `Try[URL]`. Если URL построен правильно мы вернём 
`Success[URL]`. Если конструктор URL выбросит исключение `MalformedURLException`, оно будет
обёрнуто в `Failure[A]`. 

Для эотго мы воспользовались методом `apply` из объекта-компаньона для `Try`. Этот метод
принимает значение типа `A` по имени (в данном случае URL). Для нашего примера это означает,
что `new URL(url)` будет выполнено внутри метода `apply` из объекта  `Try`. В этом методе
происходит обработка ???не фатальных (non-fatal)??? исключений, если случится исключение, то
метод вернёт `Failure`. 

Так `parseURL("http://danielwestheide.com")` вернёт `Success[URL]`, в то время как 
`parseURL("garbage")` вернёт `Failure[URL]` с исключением `MalformedURLException` внутри. 

### Использоваие значений типа `Try`

Working with Try instances is actually very similar to working with Option values, so you won’t see many surprises here.

You can check if a Try is a success by calling isSuccess on it and then conditionally retrieve the wrapped value by calling get on it. But believe me, there aren’t many situations where you will want to do that.

It’s also possible to use getOrElse to pass in a default value to be returned if the Try is a Failure:

~~~
val url = parseURL(Console.readLine("URL: ")) getOrElse new URL("http://duckduckgo.com")
~~~

If the URL given by the user is malformed, we use the URL of DuckDuckGo as a fallback.

#### Chaining operations

One of the most important characteristics of the Try type is that, like Option, it supports all the higher-order methods you know from other types of collections. As you will see in the examples to follow, this allows you to chain operations on Try values and catch any exceptions that might occur, and all that in a very readable manner.

#### Mapping and flat mapping

Mapping a Try[A] that is a Success[A] to a Try[B] results in a Success[B]. If it’s a Failure[A], the resulting Try[B] will be a Failure[B], on the other hand, containing the same exception as the Failure[A]:

~~~
parseURL("http://danielwestheide.com").map(_.getProtocol)
// results in Success("http")
parseURL("garbage").map(_.getProtocol)
// results in Failure(java.net.MalformedURLException: no protocol: garbage)
~~~

If you chain multiple map operations, this will result in a nested Try structure, which is usually not what you want. Consider this method that returns an input stream for a given URL:

~~~
import java.io.InputStream
def inputStreamForURL(url: String): Try[Try[Try[InputStream]]] = parseURL(url).map { u =>
  Try(u.openConnection()).map(conn => Try(conn.getInputStream))
}
~~~

Since the anonymous functions passed to the two map calls each return a Try, the return type is a Try[Try[Try[InputStream]]].

This is where the fact that you can flatMap a Try comes in handy. The flatMap method on a Try[A] expects to be passed a function that receives an A and returns a Try[B]. If our Try[A] instance is already a Failure[A], that failure is returned as a Failure[B], simply passing along the wrapped exception along the chain. If our Try[A] is a Success[A], flatMap unpacks the A value in it and maps it to a Try[B] by passing this value to the mapping function.

This means that we can basically create a pipeline of operations that require the values carried over in Success instances by chaining an arbitrary number of flatMap calls. Any exceptions that happen along the way are wrapped in a Failure, which means that the end result of the chain of operations is a Failure, too.

Let’s rewrite the inputStreamForURL method from the previous example, this time resorting to flatMap:

~~~
def inputStreamForURL(url: String): Try[InputStream] = parseURL(url).flatMap { u =>
  Try(u.openConnection()).flatMap(conn => Try(conn.getInputStream))
}
~~~

#### Filter and foreach

Of course, you can also filter a Try or call foreach on it. Both work exactly as you would except after having learned about Option.

The filter method returns a Failure if the Try on which it is called is already a Failure or if the predicate passed to it returns false (in which case the wrapped exception is a NoSuchElementException). If the Try on which it is called is a Success and the predicate returns true, that Succcess instance is returned unchanged:

~~~
def parseHttpURL(url: String) = parseURL(url).filter(_.getProtocol == "http")
parseHttpURL("http://apache.openmirror.de") // results in a Success[URL]
parseHttpURL("ftp://mirror.netcologne.de/apache.org") // results in a Failure[URL]
~~~

The function passed to foreach is executed only if the Try is a Success, which allows you to execute a side-effect. The function passed to foreach is executed exactly once in that case, being passed the value wrapped by the Success:

~~~
parseHttpURL("http://danielwestheide.com").foreach(println)
~~~

#### For comprehensions

The support for flatMap, map and filter means that you can also use for comprehensions in order to chain operations on Try instances. Usually, this results in more readable code. To demonstrate this, let’s implement a method that returns the content of a web page with a given URL using for comprehensions.

~~~
import scala.io.Source
def getURLContent(url: String): Try[Iterator[String]] =
  for {
    url <- parseURL(url)
    connection <- Try(url.openConnection())
    is <- Try(connection.getInputStream)
    source = Source.fromInputStream(is)
  } yield source.getLines()
~~~

There are three places where things can go wrong, all of them covered by usage of the Try type. First, the already implemented parseURL method returns a Try[URL]. Only if this is a Success[URL], we will try to open a connection and create a new input stream from it. If opening the connection and creating the input stream succeeds, we continue, finally yielding the lines of the web page. Since we effectively chain multiple flatMap calls in this for comprehension, the result type is a flat Try[Iterator[String]].

Please note that this could be simplified using Source#fromURL and that we fail to close our input stream at the end, both of which are due to my decision to keep the example focussed on getting across the subject matter at hand.

Pattern Matching
-------------------------------------------

At some point in your code, you will often want to know whether a Try instance you have received as the result of some computation represents a success or not and execute different code branches depending on the result. Usually, this is where you will make use of pattern matching. This is easily possible because both Success and Failure are case classes.

We want to render the requested page if it could be retrieved, or print an error message if that was not possible:

~~~
import scala.util.Success
import scala.util.Failure
getURLContent("http://danielwestheide.com/foobar") match {
  case Success(lines) => lines.foreach(println)
  case Failure(ex) => println(s"Problem rendering URL content: ${ex.getMessage}")
}
~~~

Recovering from a Failure
-----------------------------------------------

If you want to establish some kind of default behaviour in the case of a Failure, you don’t have to use getOrElse. An alternative is recover, which expects a partial function and returns another Try. If recover is called on a Success instance, that instance is returned as is. Otherwise, if the partial function is defined for the given Failure instance, its result is returned as a Success.

Let’s put this to use in order to print a different message depending on the type of the wrapped exception:

~~~
import java.net.MalformedURLException
import java.io.FileNotFoundException
val content = getURLContent("garbage") recover {
  case e: FileNotFoundException => Iterator("Requested page does not exist")
  case e: MalformedURLException => Iterator("Please make sure to enter a valid URL")
  case _ => Iterator("An unexpected error has occurred. We are so sorry!")
}
~~~

We could now safely get the wrapped value on the Try[Iterator[String]] that we assigned to content, because we know that it must be a Success. Calling content.get.foreach(println) would result in Please make sure to enter a valid URL being printed to the console.

Conclusion
---------------------------------------------

Idiomatic error handling in Scala is quite different from the paradigm known from languages like Java or Ruby. The Try type allows you to encapsulate computations that result in errors in a container and to chain operations on the computed values in a very elegant way. You can transfer what you know from working with collections and with Option values to how you deal with code that may result in errors – all in a uniform way.

To keep this article at a reasonable length, I haven’t explained all of the methods available on Try. Like Option, Try supports the orElse method. The transform and recoverWith methods are also worth having a look at, and I encourage you to do so.

In the next part we are going to deal with Either, an alternative type for representing computations that may result in errors, but with a wider scope of application that goes beyond error handling.










