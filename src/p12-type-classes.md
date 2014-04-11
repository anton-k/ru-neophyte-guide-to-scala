Глава 12: Классы типов
====================================

После изучения нескольких техник функционального программирования, а именно
композиция частично определённых функций, частичное применение функций, каррирование,
мы собираемся продолжить в том же духе, мы будем учиться делать код настолько гибким 
насколько это возможно.

Но на этот раз мы поговорим не функциях, а о типах Мы научимся работать с ситемой типов так, чтобы
она не мешала нам, а помогала, чтобы наш код был расширяемым. Мы собираемся узнать о *классах типов* (type class).

Вам может показаться, что это не более чем экзатичная идея, привнесённая в Scala 
фанатами Haskell, которая лишена практического применения. Но это явно не так. 
Классы типов стали важной частью стандартной библиотеки Scala, очень часто они встречаются
и в сторонних свободных библиотеках. Поэтому было бы хорошо с ними разобраться.

Я расскажу об основной идее классов типов, чем почему они так полезны, какие приемущества
они дают  пользователям нашего кода и как реализовать и применять наши собственные классы типов.

Проблема
---------------------------------------

Вместо того чтобы начать с абстрактного опредеелния, давайте попробуем разобраться что к чему
на практическом примере, пусть и весьма упрощённом.

Предположим, что мы хотим написать классную библиотеку для статистики. Это означает, что
мы собираемся написать кучу функций, который будут принимать коллекции значений и возвращать
какие-нибудь собирательные показатели.  Предположим, что мы ограничены в операциях над коллекциями.
Мы можем лишь обращаться по индексу и пользоваться методом `reduce` из стандартной библиотеки для 
коллекций. Мы накладываем эти ограничения просто потому, что так мы избавимся от лишних 
деталей, и пример станет доступным для изложения в блоге. Наконец, мы предполагаем,
что значения поступают к нам отсортированными.

Мы начнём с очень грубой реализации поиска медианы, квартилей и межквартильный интервал для
чисел типа `Double`:

~~~
object Statistics {
  def median(xs: Vector[Double]): Double = xs(xs.size / 2)

  def quartiles(xs: Vector[Double]): (Double, Double, Double) =
    (xs(xs.size / 4), median(xs), xs(xs.size / 4 * 3))

  def iqr(xs: Vector[Double]): Double = quartiles(xs) match {
    case (lowerQuartile, _, upperQuartile) => upperQuartile - lowerQuartile
  }

  def mean(xs: Vector[Double]): Double = {
    xs.reduce(_ + _) / xs.size
  }
}
~~~

Медиана разделяет данные посередине, в то время как нижний и верхний квартили (первый и третьий элементы кортежа,
который возвращает функция `quartiles`) разделяют данные на нижние и верхние 25%. Метод `iqr` возвращает 
вкличину межквартильного интервала, которая представляет собой разницу между верхним и нижни квартилями.

Вдруг нам понадобилось вычисление этих параметров не только для `Double`. Неужели мы будем снова определять 
эти методы для `Int`?

Конечно нет! Во первых мы не можем перегрузить объявленные методы для `Vector[Int]` без некоторых трюков,
потому что тип параметр страдает от стирания типов (type erasure). При этом в нашем коде появятся повторы, не так ли?

Если бы `Int` и `Double` наследовали от одного общего класса вроде `Number`! Тогда мы могли
бы написать наши методы в более общем виде:

~~~
object Statistics {
  def median(xs: Vector[Number]): Number = ???
  def quartiles(xs: Vector[Number]): (Number, Number, Number) = ???
  def iqr(xs: Vector[Number]): Number = ???
  def mean(xs: Vector[Number]): Number = ???
}
~~~

К счастью, такого трэйта нет, и мы не сможем выбрать это неверное решение. Но в других ситуациях
такая возможность моэет представиться, несмотря на то что это по-прежнему останется плохим решением.
Мы не только ослабляем наши ограничения по типам, мы закрываем интерфейс для расширения, с помощью
тех типов, которые нам пока неизвестны, мы не можем взять численный тип из сторонней библиотеки и унаследовать
его от трэйта `Number`.

В Ruby мы могли бы воспользоваться обезьяним патчем (monkey patch), засоряя глобальное пространство имён
расширением к новому типу, так мы сможем заставить его вести себя как `Number`. Разработчики Java, знакомые
с шаблонами проектирования, могут предложить воспользоваться шаблоном адаптор:

~~~
object Statistics {
  trait NumberLike[A] {
    def get: A
    def plus(y: NumberLike[A]): NumberLike[A]
    def minus(y: NumberLike[A]): NumberLike[A]
    def divide(y: Int): NumberLike[A]
  }
  case class NumberLikeDouble(x: Double) extends NumberLike[Double] {
    def get: Double = x
    def minus(y: NumberLike[Double]) = NumberLikeDouble(x - y.get)
    def plus(y: NumberLike[Double]) = NumberLikeDouble(x + y.get)
    def divide(y: Int) = NumberLikeDouble(x / y)
  }
  type Quartile[A] = (NumberLike[A], NumberLike[A], NumberLike[A])
  def median[A](xs: Vector[NumberLike[A]]): NumberLike[A] = xs(xs.size / 2)
  def quartiles[A](xs: Vector[NumberLike[A]]): Quartile[A] =
    (xs(xs.size / 4), median(xs), xs(xs.size / 4 * 3))
  def iqr[A](xs: Vector[NumberLike[A]]): NumberLike[A] = quartiles(xs) match {
    case (lowerQuartile, _, upperQuartile) => upperQuartile.minus(lowerQuartile)
  }
  def mean[A](xs: Vector[NumberLike[A]]): NumberLike[A] =
    xs.reduce(_.plus(_)).divide(xs.size)
}
~~~

Мы решили проблему расширяемости. Пользователи библиотеки могут передать даптор `NumberLike`
для `Int` (который мы скорее всего сами же и напишем) или для любого другого типа, который
может выступать в роли числа, без необходимости перекомпиляции, модуля в котором определены
методы вычисления статистических параметров.

Но постоянное заворачивание и разворачивание чисел в адапторы -- не только утомительно
для написания и чтения, но и ведёт к тому, что нам приходится создавать много значений
для адапторов.


Классы типов приходят на помощь!
---------------------------------------------------------

Классы типов предлагают наилучшее решение этой проблемы. Классы типов -- одна из основных
особенностей языка Haskell. Несмотря на название, они не имеют ничего общего с понятием 
класс из объектно ориентированного программирования.

Класс типов `C` определяет некоторое поведение в виде набора операций, которые должны
быть определена на типе `T`, для того чтобы тот был членом класса типов.
Является ли некоторый тип `T` членом класса `C` не указывается в определении класса.
Вместо этого любой пользователь может объявить свой тип членом `C`, определив на нём 
все необходимые операции. Как только `T` стал членом класса `C`, функции их класса `C`
могут вызываться на значениях типа `T`:

Классы типов реализуют ситуативный полиморфизм (ad-hoc polymorphism). Код, зависящий
от классов типов, открыт для расширений, без необходимости создания объектов-адаптеров.


### Создание классов типов

В Scala классы типов реализуются с помощью комбинации нескольких техник. 
Это более окольный путь чем в Haskell, но предлагает большую возможность для контроля.

Создание класса типов в Scala происходит в несколько шагов. Во первых, давайте определим
трэйт. Он и будет нашим классом типов:

~~~
object Math {
  trait NumberLike[T] {
    def plus(x: T, y: T): T
    def divide(x: T, y: Int): T
    def minus(x: T, y: T): T
  }
}
~~~

Мы создали класс типов с названием `NumberLike`. Классы типов всегда принимают один или несколько типов-параметров.
Обычно они не содержат состояния, то есть методы определённые в таком трэйте оперируют только данными, переданными
в метод. В то время как метод из адаптора был привязан к значению и имел один аргумент, наш
новые метод имеет два аргумента типа `T`. Значение адаптора превратилось в первый аргумент метода,
определённого в `NumberLike`.

### Определение значений по умолчанию

Второй шаг реализации класса типа заключается в определении в объекте-компаньоне значений по умолчанию, принадлежащих
нашему классу типов. Скоро мы узнаем почему хорошо так делать. Но сначала давайте сделаем это, давайте сделаем
`Double` и `Int` членами класса типов `NumberLike`:

~~~
object Math {
  trait NumberLike[T] {
    def plus(x: T, y: T): T
    def divide(x: T, y: Int): T
    def minus(x: T, y: T): T
  }

  object NumberLike {
    implicit object NumberLikeDouble extends NumberLike[Double] {
      def plus(x: Double, y: Double): Double = x + y
      def divide(x: Double, y: Int): Double = x / y
      def minus(x: Double, y: Double): Double = x - y
    }

    implicit object NumberLikeInt extends NumberLike[Int] {
      def plus(x: Int, y: Int): Int = x + y
      def divide(x: Int, y: Int): Int = x / y
      def minus(x: Int, y: Int): Int = x - y
    }
  }
}
~~~

Отметим два момента. Во первых, видно, что реализации практически одинаковы. Это не всегда так с классами типов.
Наш класс `NumberLike` очень специфичен. Сокоро мы встретимся с примерами, в которых не так много дублирования
в реалзиации методов. Во вторых, пожалуйста не заостряйте внимание на том, что мы теряем в точности при 
выполнении целочисленного деления в `NumberLikeInt`. Это просто учебный пример.

Как видно из примера, члены класса являются объектами-синглтонами. Также опбратите внимание на ключевое
слово `implicit` перед каждой из реализаций. Как раз за счёт этого мы и можем реализовать классы типов в Scala.
Это ключевое слово делает методы, определённые в синглтоне, неявно доступными при определённых условиях.
Подробнее мы разберёмся в этом в следующем разделе.

### Применение классов типов

Now that we have our type class and two default implementations for common types, we want to code against this type class in our statistics module. Let’s focus on the mean method for now:

~~~
object Statistics {
  import Math.NumberLike
  def mean[T](xs: Vector[T])(implicit ev: NumberLike[T]): T =
    ev.divide(xs.reduce(ev.plus(_, _)), xs.size)
}
~~~

This may look a little intimidating at first, but it’s actually quite simple. Our method takes a type parameter T and a single parameter of type Vector[T].

The idea to constrain a parameter to types that are members of a specific type class is realized by means of the implicit second parameter list. What does this mean? Basically, that a value of type NumberLike[T] must be implicitly available in the current scope. This is the case if an implicit value has been declared and made available in the current scope, very often by importing the package or object in which that implicit value is defined.

If and only if no other implicit value can be found, the compiler will look in the companion object of the type of the implicit parameter. Hence, as a library designer, putting your default type class implementations in the companion object of your type class trait means that users of your library can easily override these implementations with their own ones, which is exactly what you want. Users can also pass in an explicit value for an implicit parameter to override the implicit values that are in scope.

Let’s see if the default type class implementations can be resolved:

~~~
val numbers = Vector[Double](13, 23.0, 42, 45, 61, 73, 96, 100, 199, 420, 900, 3839)
println(Statistics.mean(numbers))
~~~

Wonderful! If we try this with a Vector[String], we get an error at compile time, stating that no implicit value could be found for parameter ev: NumberLike[String]. If you don’t like this error message, you can customize it by annotating your type class trait with the @implicitNotFound annotation:

~~~
object Math {
  import annotation.implicitNotFound
  @implicitNotFound("No member of type class NumberLike in scope for ${T}")
  trait NumberLike[T] {
    def plus(x: T, y: T): T
    def divide(x: T, y: Int): T
    def minus(x: T, y: T): T
  }
}
~~~

#### Context bounds

A second, implicit parameter list on all methods that expect a member of a type class can be a little verbose. As a shortcut for implicit parameters with only one type parameter, Scala provides so-called context bounds. To show how those are used, we are going to implement our other statistics methods using those instead:

~~~
object Statistics {
  import Math.NumberLike
  def mean[T](xs: Vector[T])(implicit ev: NumberLike[T]): T =
    ev.divide(xs.reduce(ev.plus(_, _)), xs.size)
  def median[T : NumberLike](xs: Vector[T]): T = xs(xs.size / 2)
  def quartiles[T: NumberLike](xs: Vector[T]): (T, T, T) =
    (xs(xs.size / 4), median(xs), xs(xs.size / 4 * 3))
  def iqr[T: NumberLike](xs: Vector[T]): T = quartiles(xs) match {
    case (lowerQuartile, _, upperQuartile) =>
      implicitly[NumberLike[T]].minus(upperQuartile, lowerQuartile)
  }
}
~~~

A context bound T : NumberLike means that an implicit value of type NumberLike[T] must be available, and so is really equivalent to having a second implicit parameter list with a NumberLike[T] in it. If you want to access that implicitly available value, however, you need to call the implicitly method, as we do in the iqr method. If your type class requires more than one type parameter, you cannot use the context bound syntax.

### Custom type class members

As a user of a library that makes use of type classes, you will sooner or later have types that you want to make members of those type classes. For instance, we might want to use the statistics library for instances of the Joda Time Duration type. To do that, we need Joda Time on our classpath, of course:

~~~
libraryDependencies += "joda-time" % "joda-time" % "2.1"

libraryDependencies += "org.joda" % "joda-convert" % "1.3"
~~~

Now we just have to create an implicitly available implementation of NumberLike (please make sure you have Joda Time on your classpath when trying this out):

~~~
object JodaImplicits {
  import Math.NumberLike
  import org.joda.time.Duration
  implicit object NumberLikeDuration extends NumberLike[Duration] {
    def plus(x: Duration, y: Duration): Duration = x.plus(y)
    def divide(x: Duration, y: Int): Duration = Duration.millis(x.getMillis / y)
    def minus(x: Duration, y: Duration): Duration = x.minus(y)
  }
}
~~~

If we import the package or object containing this NumberLike implementation, we can now compute the mean value for a bunch of durations:

~~~
import Statistics._
import JodaImplicits._
import org.joda.time.Duration._

val durations = Vector(standardSeconds(20), standardSeconds(57), standardMinutes(2),
  standardMinutes(17), standardMinutes(30), standardMinutes(58), standardHours(2),
  standardHours(5), standardHours(8), standardHours(17), standardDays(1),
  standardDays(4))
println(mean(durations).getStandardHours)
~~~

Use cases
--------------------------------------

Our NumberLike type class was a nice exercise, but Scala already ships with the Numeric type class, which allows you to call methods like sum or product on collections for whose type T a Numeric[T] is available. Another type class in the standard library that you will use a lot is Ordering, which allows you to provide an implicit ordering for your own types, available to the sort method on Scala’s collections.

There are more type classes in the standard library, but not all of them are ones you have to deal with on a regular basis as a Scala developer.

A very common use case in third-party libraries is that of object serialization and deserialization, most notably to and from JSON. By making your classes members of an appropriate formatter type class, you can customize the way your classes are serialized to JSON, XML or whatever format is currently the new black.

Mapping between Scala types and ones supported by your database driver is also commonly made customizable and extensible via type classes.

Summary
-------------------------------------------

Once you start to do some serious work with Scala, you will inevitably stumble upon type classes. I hope that after reading this article, you are prepared to take advantage of this powerful technique.

Scala type classes allow you to develop your Scala code in such a way that it’s open for retroactive extension while retaining as much concrete type information as possible. In contrast to approaches from other languages, they give developers full control, as default type class implementations can be overridden without much hassle, and type classes implementations are not made available in the global namespace.

You will see that this technique is especially useful when writing libraries intended to be used by others, but type classes also have their use in application code to decrease coupling between modules.


----------------------------------------------------

* <= [Глава 11: Каррирование и частичное применение](https://github.com/anton-k/ru-neophyte-guide-to-scala/blob/master/src/p11-currying.md)

* => [Глава 13: Path-dependent types](https://github.com/anton-k/ru-neophyte-guide-to-scala/blob/master/src/p13-path-dep-types.md)

* [Содержание](https://github.com/anton-k/ru-neophyte-guide-to-scala#%D0%9F%D1%83%D1%82%D0%B5%D0%B2%D0%BE%D0%B4%D0%B8%D1%82%D0%B5%D0%BB%D1%8C-%D0%BD%D0%B5%D0%BE%D1%84%D0%B8%D1%82%D0%B0-%D0%BF%D0%BE-scala)