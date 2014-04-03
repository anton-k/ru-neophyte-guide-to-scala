Part 11: Currying and Partially Applied Functions
===================================================================

Last week’s article was all about avoiding code duplication, either by lifting existing functions to match your new requirements or by composing them. In this article, we are going to have a look at two other mechanisms the Scala language provides in order to enable you to reuse your functions: Partial application of functions and currying.

Partially applied functions
----------------------------------------------------------

Scala, like many other languages following the functional programming paradigm, allows you to apply a function partially. What this means is that, when applying the function, you do not pass in arguments for all of the parameters defined by the function, but only for some of them, leaving the remaining ones blank. What you get back is a new function whose parameter list only contains those parameters from the original function that were left blank.

Do not confuse partially applied functions with partially defined functions, which are represented by the PartialFunction type in Scala.

To illustrate how partial function application works, let’s revisit our example from last week: For our imaginary free mail service, we wanted to allow the user to configure a filter so that only emails meeting certain criteria would show up in their inbox, with all others being blocked.

Our Email case class still looks like this:

~~~
case class Email(
  subject: String,
  text: String,
  sender: String,
  recipient: String)
type EmailFilter = Email => Boolean
~~~

The criteria for filtering emails were represented by a predicate Email => Boolean, which we aliased to the type EmailFilter, and we were able to generate new predicates by calling appropriate factory functions.

Two of the factory functions from last week’s article created EmailFiter instances that checked if the email text satisfied a given minimum or maximum length. This time we want to make use of partial function application to implement these factory functions. We want to have a general method sizeConstraint and be able to create more specific size constraints by fixing some of its parameters.

Here is our revised sizeConstraint method:

~~~
type IntPairPred = (Int, Int) => Boolean
def sizeConstraint(pred: IntPairPred, n: Int, email: Email) = pred(email.text.size, n)
~~~

We also define an alias IntPairPred for the type of predicate that checks a pair of integers (the value n and the text size of the email) and returns whether the email text size is okay, given n.

Note that unlike our sizeConstraint function from last week, this one does not return a new EmailFilter predicate, but simply evaluates all the arguments passed to it, returning a Boolean. The trick is to get such a predicate of type EmailFilter by partially applying sizeConstraint.

First, however, because we take the DRY principle very seriously, let’s define all the commonly used instances of IntPairPred. Now, when we call sizeConstraint, we don’t have to repeatedly write the same anonymous functions, but can simply pass in one of those:

~~~
val gt: IntPairPred = _ > _
val ge: IntPairPred = _ >= _
val lt: IntPairPred = _ < _
val le: IntPairPred = _ <= _
val eq: IntPairPred = _ == _
~~~

Finally, we are ready to do some partial application of the sizeConstraint function, fixing its first parameter with one of our IntPairPred instances:

~~~
val minimumSize: (Int, Email) => Boolean = sizeConstraint(ge, _: Int, _: Email)
val maximumSize: (Int, Email) => Boolean = sizeConstraint(le, _: Int, _: Email)
~~~

As you can see, you have to use the placeholder _ for all parameters not bound to an argument value. Unfortunately, you also have to specify the type of those arguments, which makes partial function application in Scala a bit tedious.

The reason is that the Scala compiler cannot infer these types, at least not in all cases – think of overloaded methods where it’s impossible for the compiler to know which of them you are referring to.

On the other hand, this makes it possible to bind or leave out arbitrary parameters. For example, we can leave out the first one and pass in the size to be used as a constraint:

~~~
val constr20: (IntPairPred, Email) => Boolean = sizeConstraint(_: IntPairPred, 20, _: Email)
val constr30: (IntPairPred, Email) => Boolean = sizeConstraint(_: IntPairPred, 30, _: Email)
~~~

Now we have two functions that take an IntPairPred and an Email and compare the email text size to 20 and 30, respectively, but the comparison logic has not been specified yet, as that’s exactly what the IntPairPred is good for.

This shows that, while quite verbose, partial function application in Scala is a little more flexible than in Clojure, for example, where you have to pass in arguments from left to right, but can’t leave out any in the middle.

### From methods to function objects

When doing partial application on a method, you can also decide to not bind any parameters whatsoever. The parameter list of the returned function object will be the same as for the method. You have effectively turned a method into a function that can be assigned to a val or passed around:

~~~
val sizeConstraintFn: (IntPairPred, Int, Email) => Boolean = sizeConstraint _
~~~

### Producing those EmailFilters

We still haven’t got any functions that adhere to the EmailFilter type or that return new predicates of that type – like sizeConstraint, minimumSize and maximumSize don’t return a new predicate, but a Boolean, as their signature clearly shows.

However, our email filters are just another partial function application away. By fixing the integer parameter of minimumSize and maximumSize, we can create new functions of type EmailFilter:

~~~
val min20: EmailFilter = minimumSize(20, _: Email)
val max20: EmailFilter = maximumSize(20, _: Email)
~~~

Of course, we could achieve the same by partially applying our constr20 we created above:

~~~
val min20: EmailFilter = constr20(ge, _: Email)
val max20: EmailFilter = constr20(le, _: Email)
~~~

Spicing up your functions
----------------------------------------------

Maybe you find partial function application in Scala a little too verbose, or simply not very elegant to write and look at. Lucky you, because there is an alternative.

As you should know by now, methods in Scala can have more than one parameter list. Let’s define our sizeConstraint method such that each parameter is in its own parameter list:

~~~
def sizeConstraint(pred: IntPairPred)(n: Int)(email: Email): Boolean =
  pred(email.text.size, n)
~~~

If we turn this into a function object that we can assign or pass around, the signature of that function looks like this:

~~~
val sizeConstraintFn: IntPairPred => Int => Email => Boolean = sizeConstraint _
~~~

Such a chain of one-parameter functions is called a curried function, so named after Haskell Curry, who re-discovered this technique and got all the fame for it. In fact, in the Haskell programming language, all functions are in curried form by default.

In our example, it takes an IntPairPred and returns a function that takes an Int and returns a new function. That last function, finally, takes an Email and returns a Boolean.

Now, if we we want to bind the IntPairPred, we simply apply sizeConstraintFn, which takes exactly this one parameter and returns a new one-parameter function:

~~~
val minSize: Int => Email => Boolean = sizeConstraint(ge)
val maxSize: Int => Email => Boolean = sizeConstraint(le)
~~~

There is no need to use any placeholders for parameters left blank, because we are in fact not doing any partial function application.

We can now create the same EmailFilter predicates as we did using partial function application before, by applying these curried functions:

~~~
val min20: Email => Boolean = minSize(20)
val max20: Email => Boolean = maxSize(20)
~~~

Of course, it’s possible to do all this in one step if you want to bind several parameters at once. It just means that you immediately apply the function that was returned from the first function application, without assigning it to a val first:

~~~
val min20: Email => Boolean = sizeConstraintFn(ge)(20)
val max20: Email => Boolean = sizeConstraintFn(le)(20)
~~~

### Currying existing functions

It’s not always the case the you know beforehand whether writing your functions in curried form makes sense or not – after all, the usual function application looks a little more verbose than for functions that have only declared a single parameter list for all their parameters. Also, you’ll sometimes want to work with third-party functions in curried form, when they are written with a single parameter list.

Transforming a function with multiple parameters in one list to curried form is, of course, just another higher-order function, generating a new function from an existing one. This transformation logic is available in form of the curried method on functions with more than one parameter. Hence, if we have a function sum, taking two parameters, we can get the curried version simply calling calling its curried method:

~~~
val sum: (Int, Int) => Int = _ + _
val sumCurried: Int => Int => Int = sum.curried
~~~

If you need to do the reverse, you have Function.uncurried at your fingertips, which you need to pass the curried function to get it back in uncurried form.

### Injecting your dependencies the functional way

To close this article, let’s have a look at what role curried functions can play in the large. If you come from the enterprisy Java or .NET world, you will be very familiar with the necessity to use more or less fancy dependency injection containers that take the heavy burden of providing all your objects with their respective dependencies off you. In Scala, you don’t really need any external tool for that, as the language already provides numerous features that make it much less of a pain to this all on your own.

When programming in a very functional way, you will notice that there is still a need to inject dependencies: Your functions residing in the higher layers of your application will have to make calls to other functions. Simply hard-coding the functions to call in the body of your functions makes it difficult to test them in isolation. Hence, you will need to pass the functions your higher-layer function depends on as arguments to that function.

It would not be DRY at all to always pass the same dependencies to your function when calling it, would it? Curried functions to the rescue! Currying and partial function application are one of several ways of injecting dependencies in functional programming.

The following, very simplified example illustrates this technique:

~~~
case class User(name: String)
trait EmailRepository {
  def getMails(user: User, unread: Boolean): Seq[Email]
}
trait FilterRepository {
  def getEmailFilter(user: User): EmailFilter
}
trait MailboxService {
  def getNewMails(emailRepo: EmailRepository)(filterRepo: FilterRepository)(user: User) =
    emailRepo.getMails(user, true).filter(filterRepo.getEmailFilter(user))
  val newMails: User => Seq[Email]
}
~~~

We have a service that depends on two different repositories, These dependencies are declared as parameters to the getNewMails method, each in their own parameter list.

The MailboxService already has a concrete implementation of that method, but is lacking one for the newMails field. The type of that field is User => Seq[Email] – that’s the function that components depending on the MailboxService will call.

We need an object that extends MailboxService. The idea is to implement newMails by currying the getNewMails method and fixing it with concrete implementations of the dependencies, EmailRepository and FilterRepository:

~~~
object MockEmailRepository extends EmailRepository {
  def getMails(user: User, unread: Boolean): Seq[Email] = Nil
}
object MockFilterRepository extends FilterRepository {
  def getEmailFilter(user: User): EmailFilter = _ => true
}
object MailboxServiceWithMockDeps extends MailboxService {
  val newMails: (User) => Seq[Email] =
    getNewMails(MockEmailRepository)(MockFilterRepository) _
}
~~~

We can now call MailboxServiceWithMoxDeps.newMails(User("daniel")) without having to specify the two repositories to be used. In a real application, of course, we would very likely not use a direct reference to a concrete implementation of the service, but have this injected, too.

This is probably not the most powerful and scaleable way of injecting your dependencies in Scala, but it’s good to have this in your tool belt, and it’s a very good example of the benefits that partial function application and currying can provide in the wild. If you want know more about this, I recommend to have a look at the excellent slides for the presentation ”Dependency Injection in Scala” by Debasish Ghosh, which is also where I first came across this technique.

Summary
------------------------------------------

In this article, we discussed two additional functional programming techniques that help you keep your code free of duplication and, on top of that, give you a lot of flexibility, allowing you to reuse your functions in manifold ways. Partial function application and currying have more or less the same effect, but sometimes one of them is the more elegant solution.

In the next part of this series, we will continue to look at ways to keep things flexible, discussing the what and how of type classes in Scala.




