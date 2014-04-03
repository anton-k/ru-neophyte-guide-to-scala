Part 7: The Either Type
================================================

In the previous article, I discussed functional error handling using Try, which was introduced in Scala 2.10. I also mentioned the existence of another, somewhat similar type called Either, which is the subject of this article. You will learn how to use it, when to use it, and what its particular pitfalls are.

Speaking of which, at least at the time of this writing, Either has some serious design flaws you need to be aware of, so much so that one might argue about whether to use it at all. So why then should you learn about Either at all?

For one, people will not all migrate their existing code bases to use Try for dealing with exceptions, so it is good to be able to understand the intricacies of this type, too.

Moreover, Try is not really an all-out replacement for Either, only for one particular usage of it, namely handling exceptions in a functional way. As it stands, Try and Either really complement each other, each covering different use cases. And, as flawed as Either may be, in certain situations it will still be a very good fit.

The semantics
-----------------------------------------

Like Option and Try, Either is a container type. Unlike the aforementioned types, it takes not only one, but two type parameters: An Either[A, B] instance can contain either an instance of A, or an instance of B. This is different from a Tuple2[A, B], which contains both an A and a B instance.

Either has exactly two sub types, Left and Right. If an Either[A, B] object contains an instance of A, then the Either is a Left. Otherwise it contains an instance of B and is a Right.

There is nothing in the semantics of this type that specifies one or the other sub type to represent an error or a success, respectively. In fact, Either is a general-purpose type for use whenever you need to deal with situations where the result can be of one of two possible types. Nevertheless, error handling is a popular use case for it, and by convention, when using it that way, the Left represents the error case, whereas the Right contains the success value.


Creating an Either
------------------------------------------

Creating an instance of Either is trivial. Both Left and Right are case classes, so if we want to implement a rock-solid internet censorship feature, we can just do the following:

~~~
import scala.io.Source
import java.net.URL
def getContent(url: URL): Either[String, Source] =
  if (url.getHost.contains("google"))
    Left("Requested URL is blocked for the good of the people!")
  else
    Right(Source.fromURL(url))
~~~

Now, if we call getContent(new URL("http://danielwestheide.com")), we will get a scala.io.Source wrapped in a Right. If we pass in new URL("https://plus.google.com"), the return value will be a Left containing a String.

Working with Either values
----------------------------------------------

Some of the very basic stuff works just as you know from Option or Try: You can ask an instance of Either if it isLeft or isRight. You can also do pattern matching on it, which is one of the most familiar and convenient ways of working with objects of this type:

~~~
getContent(new URL("http://google.com")) match {
  case Left(msg) => println(msg)
  case Right(source) => source.getLines.foreach(println)
}
~~~

### Projections

You cannot, at least not directly, use an Either instance like a collection, the way you are familiar with from Option and Try. This is because Either is designed to be unbiased.

Try is success-biased: it offers you map, flatMap and other methods that all work under the assumption that the Try is a Success, and if that’s not the case, they effectively don’t do anything, returning the Failure as-is.

The fact that Either is unbiased means that you first have to choose whether you want to work under the assumption that it is a Left or a Right. By calling left or right on an Either value, you get a LeftProjection or RightProjection, respectively, which are basically left- or right-biased wrappers for the Either.

#### Mapping

Once you have a projection, you can call map on it:

~~~
val content: Either[String, Iterator[String]] =
  getContent(new URL("http://danielwestheide.com")).right.map(_.getLines())
// content is a Right containing the lines from the Source returned by getContent
val moreContent: Either[String, Iterator[String]] =
  getContent(new URL("http://google.com")).right.map(_.getLines)
// moreContent is a Left, as already returned by getContent
~~~

Regardless of whether the Either[String, Source] in this example is a Left or a Right, it will be mapped to an Either[String, Iterator[String]]. If it’s called on a Right, the value inside it will be transformed. If it’s a Left, that will be returned unchanged.

We can do the same with a LeftProjection, of course:

~~~
val content: Either[Iterator[String], Source] =
  getContent(new URL("http://danielwestheide.com")).left.map(Iterator(_))
// content is the Right containing a Source, as already returned by getContent
val moreContent: Either[Iterator[String], Source] =
  getContent(new URL("http://google.com")).left.map(Iterator(_))
// moreContent is a Left containing the msg returned by getContent in an Iterator
~~~

Now, if the Either is a Left, its wrapped value is transformed, whereas a Right would be returned unchanged. Either way, the result is of type Either[Iterator[String], Source].

Please note that the map method is defined on the projection types, not on Either, but it does return a value of type Either, not a projection. In this, Either deviates from the other container types you know. The reason for this has to do with Either being unbiased, but as you will see, this can lead to some very unpleasant problems in certain cases. It also means that if you want to chain multiple calls to map, flatMap and the like, you always have to ask for your desired projection again in between.

#### Flat mapping

Projections also support flat mapping, avoiding the common problem of creating a convoluted structure of multiple inner and outer Either types that you will end up with if you nest multiple calls to map.

I’m putting very high requirements on your suspension of disbelief now, coming up with a completely contrived example. Let’s say we want to calculate the average number of lines of two of my articles. You’ve always wanted to do that, right? Here’s how we could solve this challenging problem:

~~~
val part5 = new URL("http://t.co/UR1aalX4")
val part6 = new URL("http://t.co/6wlKwTmu")
val content = getContent(part5).right.map(a =>
  getContent(part6).right.map(b =>
    (a.getLines().size + b.getLines().size) / 2))
~~~

What we’ll end up with is an Either[String, Either[String, Int]]. Now, content being a nested structure of Rights, we could flatten it by calling the joinRight method on it (you also have joinLeft available to flatten a nested structure of Lefts).

However, we can avoid creating this nested structure altogether. If we flatMap on our outer RightProjection, we get a more pleasant result type, unpacking the Right of the inner Either:

~~~
val content = getContent(part5).right.flatMap(a =>
  getContent(part6).right.map(b =>
    (a.getLines().size + b.getLines().size) / 2))
~~~

Now content is a flat Either[String, Int], which makes it a lot nicer to work with, for example using pattern matching.

#### For comprehensions

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








