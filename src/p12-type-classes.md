Part 12: Type Classes
====================================

After having discussed several functional programming techniques for keeping things DRY and flexible in the last two weeks, in particular function composition, partial function application, and currying, we are going to stick with the general notion of making your code as flexible as possible.

However, this time, we are not looking so much at how you can leverage functions as first-class objects to achieve this goal. Instead, this article is all about using the type system in such a manner that it’s not in the way, but rather supports you in keeping your code extensible: You’re going to learn about type classes.

You might think that this is some exotic idea without practical relevance, brought into the Scala community by some vocal Haskell fanatics. This is clearly not the case. Type classes have become an important part of the Scala standard library and even more so of many popular and commonly used third-party open-source libraries, so it’s generally a good idea to make yourself familiar with them.

I will discuss the idea of type classes, why they are useful, how to benefit from them as a client, and how to implement your own type classes and put them to use for great good.

The problem
---------------------------------------

Instead of starting off by giving an abstract explanation of what type classes are, let’s tackle this subject by means of an – admittedly simplified, but nevertheless resonably practical – example.

Imagine that we want to write a fancy statistics library. This means we want to provide a bunch of functions that operate on collections of numbers, mostly to compute some aggregate values for them. Imagine further that we are restricted to accessing an element from such a collection by index and to using the reduce method defined on Scala collections. We impose this restriction on ourselves because we are going to re-implement a little bit of what the Scala standard library already provides – simply because it’s a nice example without many distractions, and it’s small enough for a blog post. Finally, our implementation assumes that the values we get are already sorted.

We will start with a very crude implementation of median, quartiles, and iqr numbers of type Double:

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

The median cuts a data set in half, whereas the lower and upper quartile (first and third element of the tuple returned by our quartile method), split the lowest and highest 25 percent of the data, respectively. Our iqr method returns the interquartile range, which is the difference between the upper and lower quartile.

Now, of course, we want to support more than just double numbers. So let’s implement all these methods again for Int numbers, right?

Well, no! First of all, we can’t simply overload the methods defined above for Vector[Int] without some dirty tricks, because the type parameter suffers from type erasure. Also, that would be a tiny little bit repetitious, wouldn’t it?

If only Int and Double would extend from a common base class or implement a common trait like Number! We could be tempted to change the type required and returned by our methods to that more general type. Our method signatures would look like this:

~~~
object Statistics {
  def median(xs: Vector[Number]): Number = ???
  def quartiles(xs: Vector[Number]): (Number, Number, Number) = ???
  def iqr(xs: Vector[Number]): Number = ???
  def mean(xs: Vector[Number]): Number = ???
}
~~~

Thankfully, in this case there is no such common trait, so we aren’t tempted to walk this road at all. However, in other cases, that might very well be the case – and still be a bad idea. Not only do we drop previously available type information, we also close our API against future extensions to types whose sources we don’t control: We cannot make some new number type coming from a third party extend the Number trait.

Ruby’s answer to that problem is monkey patching, polluting the global namespace with an extension to that new type, making it act like a Number after all. Java developers who have got beaten up by the Gang of Four in their youth, on the other hand, will think that an Adapter may solve all of their problems:

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

Now we have solved the problem of extensibility: Users of our library can pass in a NumberLike adapter for Int (which we would likely provide ourselves) or for any possible type that might behave like a number, without having to recompile the module in which our statistics methods are implemented.

However, always wrapping your numbers in an adapter is not only tiresome to write and read, it also means that you have to create a lot of instances of your adapter classes when interacting with our library.

Type classes to the rescue!
---------------------------------------------------------

A powerful alternative to the approaches outlined so far is, of course, to define and use a type class. Type classes, one of the prominent features of the Haskell language, despite their name, haven’t got anything to do with classes in object-oriented programming.

A type class C defines some behaviour in the form of operations that must be supported by a type T for it to be a member of type class C. Whether the type T is a member of the type class C is not inherent in the type. Rather, any developer can declare that a type is a member of a type class simply by providing implementations of the operations the type must support. Now, once T is made a member of the type class C, functions that have constrained one or more of their parameters to be members of C can be called with arguments of type T.

As such, type classes allow ad-hoc and retroactive polymorphism. Code that relies on type classes is open to extension without the need to create adapter objects.

### Creating a type class

In Scala, type classes can be implemented and used by a combination of techniques. It’s a little more involved than in Haskell, but also gives developers more control.

Creating a type class in Scala involves several steps. First, let’s define a trait. This is the actual type class:

~~~
object Math {
  trait NumberLike[T] {
    def plus(x: T, y: T): T
    def divide(x: T, y: Int): T
    def minus(x: T, y: T): T
  }
}
~~~

We have created a type class called NumberLike. Type classes always take one or more type parameters, and they are usually designed to be stateless, i.e. the methods defined on our NumberLike trait operate only on the passed in arguments. In particular, where our adapter above operated on its member of type T and one argument, the methods defined for our NumberLike type class take two parameters of type T each – the member has become the first parameter of the operations supported by NumberLike.

### Providing default members

The second step in implementing a type class is usually to provide some default implementations of your type class trait in its companion object. We will see in a moment why this is generally a good strategy. First, however, let’s do this, too, by making Double and Int members of our NumberLike type class:

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

Two things: First, you see that the two implementations are basically identical. That is not always the case when creating members of a type classes. Our NumberLike type class is just a rather narrow domain. Later in the article, I will give examples of type classes where there is a lot less room for duplication when implementing them for multiple types. Second, please ignore the fact that we are losing precision in NumberLikeInt by doing integer division. It’s all to keep things simple for this example.

As you can see, members of type classes are usually singleton objects. Also, please note the implicit keyword before each of the type class implementations. This is one of the crucial elements for making type classes possible in Scala, making type class members implicitly available under certain conditions. More about that in the next section.

### Coding against type classes

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