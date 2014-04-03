Part 9: Promises and Futures in Practice
===================================================

In the previous article in this series, I introduced you to the Future type, its underlying paradigm, and how to put it to use to write highly readable and composable asynchronously executing code.

In that article, I also mentioned that Future is really only one piece of the puzzle: It’s a read-only type that allows you to work with the values it will compute and handle failure to do so in an elegant way. In order for you to be able to read a computed value from a Future, however, there must be a way for some other part of your software to put that value there. In this post, I will show you how this is done by means of the Promise type, followed by some guidance on how to use futures and promises in practice.

Promises
------------------------------------------

In the previous article on futures, we had a sequential block of code that we passed to the apply method of the Future companion object, and, given an ExecutionContext was in scope, it magically executed that code block asynchronously, returning its result as a Future.

While this is an easy way to get a Future when you want one, there is an alternative way to create Future instances and have them complete with a success or failure. Where Future provides an interface exclusively for querying, Promise is a companion type that allows you to complete a Future by putting a value into it. This can be done exactly once. Once a Promise has been completed, it’s not possible to change it any more.

A Promise instance is always linked to exactly one instance of Future. If you call the apply method of Future again in the REPL, you will indeed notice that the Future returned is a Promise, too:

~~~
import concurrent.Future
import concurrent.ExecutionContext.Implicits.global
val f: Future[String] = Future { "Hello world!" }
// REPL output: 
// f: scala.concurrent.Future[String] = scala.concurrent.impl.Promise$DefaultPromise@793e6657
~~~

The object you get back is a DefaultPromise, which implements both Future and Promise. This is an implementation detail, however. The Future and the Promise to which it belongs may very well be separate objects.

What this little example shows is that there is obviously no way to complete a Future other than through a Promise – the apply method on Future is just a nice helper function that shields you from this.

Now, let’s see how we can get our hands dirty and work with the Promise type directly.

Promising a rosy future
--------------------------------------------------

When talking about promises that may be fulfilled or not, an obvious example domain is that of politicians, elections, campaign pledges, and legislative periods.

Suppose the politicians that then got elected into office promised their voters a tax cut. This can be represented as a Promise[TaxCut], which you can create by calling the apply method on the Promise companion object, like so:

~~~
import concurrent.Promise
case class TaxCut(reduction: Int)
// either give the type as a type parameter to the factory method:
val taxcut = Promise[TaxCut]()
// or give the compiler a hint by specifying the type of your val:
val taxcut2: Promise[TaxCut] = Promise()
~~~

Once you have created a Promise, you can get the Future belonging to it by calling the future method on the Promise instance:

~~~
val taxcutF: Future[TaxCut] = taxcut.future
~~~

The returned Future might not be the same object as the Promise, but calling the future method of a Promise multiple times will definitely always return the same object to make sure the one-to-one relationship between a Promise and its Future is preserved.

Completing a Promise
------------------------------------------------

Once you have made a Promise and told the world that you will deliver on it in the forseeable Future, you better do your very best to make it happen.

In Scala, you can complete a Promise either with a success or a failure.

#### Delivering on your Promise

To complete a Promise with a success, you call its success method, passing it the value that the Future associated with it is supposed to have:

~~~
taxcut.success(TaxCut(20))
~~~

Once you have done this, that Promise instance is no longer writable, and future attempts to do so will lead to an exception.

Also, completing your Promise like this leads to the successful completion of the associated Future. Any success or completion handlers on that future will now be called, or if, for instance, you are mapping that future, the mapping function will now be executed.

Usually, the completion of the Promise and the processing of the completed Future will not happen in the same thread. It’s more likely that you create your Promise, start computing its value in another thread and immediately return the uncompleted Future to the caller.

To illustrate, let’s do something like that for our taxcut promise:

~~~
object Government {
  def redeemCampaignPledge(): Future[TaxCut] = {
    val p = Promise[TaxCut]()
    Future {
      println("Starting the new legislative period.")
      Thread.sleep(2000)
      p.success(TaxCut(20))
      println("We reduced the taxes! You must reelect us!!!!1111")
    }
    p.future
  }
}
~~~

Please don’t get confused by the usage of the apply method of the Future companion object in this example. I’m just using it because it is so convenient for executing a block of code asynchronously. I could just as well have implemented the computation of the result (which involves a lot of sleeping) in a Runnable that is executed asynchrously by an ExecutorService, with a lot more boilerplate code. The point is that the Promise is not completed in the caller thread.

Let’s redeem our campaign pledge then and add an onComplete callback function to our future:

~~~
import scala.util.{Success, Failure}
val taxCutF: Future[TaxCut] = Government.redeemCampaignPledge()
  println("Now that they're elected, let's see if they remember their promises...")
  taxCutF.onComplete {
    case Success(TaxCut(reduction)) =>
      println(s"A miracle! They really cut our taxes by $reduction percentage points!")
    case Failure(ex) =>
      println(s"They broke their promises! Again! Because of a ${ex.getMessage}")
  }
~~~

If you try this out multiple times, you will see that the order of the console output is not deterministic. Eventually, the completion handler will be executed and run into the success case.

#### Breaking Promises like a sir

As a politician, you are pretty much used to not keeping your promises. As a Scala developer, you sometimes have no other choice, either. If that happens, you can still complete your Promise instance gracefully, by calling its failure method and passing it an exception:

~~~
case class LameExcuse(msg: String) extends Exception(msg)
object Government {
  def redeemCampaignPledge(): Future[TaxCut] = {
       val p = Promise[TaxCut]()
       Future {
         println("Starting the new legislative period.")
         Thread.sleep(2000)
         p.failure(LameExcuse("global economy crisis"))
         println("We didn't fulfill our promises, but surely they'll understand.")
       }
       p.future
     }
}
~~~

This implementation of the redeemCampaignPledge() method will to lots of broken promises. Once you have completed a Promise with the failure method, it is no longer writable, just as is the case with the success method. The associated Future will now be completed with a Failure, too, so the callback function above would run into the failure case.

If you already have a Try, you can also complete a Promise by calling its complete method. If the Try is a Success, the associated Future will be completed successfully, with the value inside the Success. If it’s a Failure, the Future will completed with that failure.

Future-based programming in practice
----------------------------------------------------------

If you want to make use of the future-based paradigm in order to increase the scalability of your application, you have to design your application to be non-blocking from the ground-up, which basically means that the functions in all your application layers are asynchronous and return futures.

A likely use case these days is that of developing a web application. If you are using a modern Scala web framework, it will allow you to return your responses as something like a Future[Response] instead of blocking and then returning your finished Response. This is important since it allows your web server to handle a huge number of open connections with a relatively low number of threads. By always giving your web server Future[Response], you maximize the utilization of the web server’s dedicated thread pool.

In the end, a service in your application might make multiple calls to your database layer and/or some external web service, receiving multiple futures, and then compose them to return a new Future, all in a very readable for comprehension, such as the one you saw in the previous article. The web layer will turn such a Future into a Future[Response].

However, how do you implement this in practice? There are are three different cases you have to consider:

### Non-blocking IO

Your application will most certainly involve a lot of IO. For example, your web application will have to talk to a database, and it might act as a client that is calling other web services.

If at all possible, make use of libraries that are based on Java’s non-blocking IO capabilities, either by using Java’s NIO API directly or through a library like Netty. Such libraries, too, can serve many connections with a reasonably-sized dedicated thread pool.

Developing such a library yourself is one of the few places where directly working with the Promise type makes a lot of sense.

### Blocking IO

Sometimes, there is no NIO-based library available. For instance, most database drivers you’ll find in the Java world nowadays are using blocking IO. If you made a query to your database with such a driver in order to respond to a HTTP request, that call would be made on a web server thread. To avoid that, place all the code talking to the database inside a future block, like so:

~~~
// get back a Future[ResultSet] or something similar:
Future {
  queryDB(query)
}
~~~

So far, we always used the implicitly available global ExecutionContext to execute such future blocks. It’s probably a good idea to create a dedicated ExecutionContext that you will have in scope in your database layer.

You can create an ExecutionContext from a Java ExecutorService, which means you will be able to tune the thread pool for executing your database calls asynchronously independently from the rest of your application:


~~~
import java.util.concurrent.Executors
import concurrent.ExecutionContext
val executorService = Executors.newFixedThreadPool(4)
val executionContext = ExecutionContext.fromExecutorService(executorService)
~~~

### Long-running computations

Depending on the nature of your application, it will occasionally have to call long-running tasks that don’t involve any IO at all, which means they are CPU-bound. These, too, should not be executed by a web server thread. Hence, you should turn them into Futures, too:

~~~
Future {
  longRunningComputation(data, moreData)
}
~~~

Again, if you have long-running computations, having them run in a separate ExecutionContext for CPU-bound tasks is a good idea. How to tune your various thread pools is highly dependent on your individual application and beyond the scope of this article.

Summary
--------------------------------------


In this article, we looked at promises, the writable part of the future-based concurrency paradigm, and how to use them to complete a Future, followed by some advice on how to put futures to use in practice.

In the next part of this series, we are taking a step back from concurrency issues and examine how functional programming in Scala can help you to make your code more reusable, a claim that has long been associated with object-oriented programming.




