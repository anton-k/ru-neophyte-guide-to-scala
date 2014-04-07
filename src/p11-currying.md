Глава 11: Каррирование и частичное применение функций
===================================================================

Прошлой статья была посвящена вопросам устранения дублирования. Для устранения 
повторов мы либо переписывали функции по-другому (определяли более общии функции),
либо выражали одни функции через композицию других. В этой статье мы посмотрим на
две новые возможности языка Scala, которые также позволят нам существенно снизить
дублирование кода. Это частичное применение функций и каррирование.

Частичное применение функций
----------------------------------------------------------

В Scala как и во многих других языках, поддерживающих функциональное программирование, мы можем
применять функцию частично. Это означает, что при вызове функции мы передаём ей лишь часть параметров,
оставшиеся же будут пустыми. При этом мы получим новую функцию, которая будет принимать те аргументы,
которые мы оставили пустыми. 

Не путайте частичное применение функций с частично определёнными функциями (типом `PartialFunction`).

Давайте посмотрим как это работает на примере. Для этого вернёмся к примеру из прошлой статье.
Для нашего воображаемого почтового сервиса мы хотели чтобы пользователь мог настраивать фильтрацию
входящих сообщений, так чтобы он получал лишь письма, удовлетворяющие некоторым требованиям.

Наш `case`-класс `Email` остаётся прежним:

~~~
case class Email(
  subject: String,
  text: String,
  sender: String,
  recipient: String)

type EmailFilter = Email => Boolean
~~~

Критерий фильтрации писем описывается предикатом `Email => Boolean`, мы дали ему синоним `EmailFilter`. 
Мы могли определять новые предикаты через уже определённые методы-фабрики.

Два метода из прошлой статьи создавали фильтры на основе проверки максимальной и минимальной длины письма.
На этот раз мы хотим воспользоваться частичным применением функции для определения этих методов. 
Мы хотим создать обобщённый метод `sizeConstraint`, так чтобы частные фильтры получались с помощью
подстановки части его параметров.  

Вот наш метод `sizeConstraint`:

~~~
type IntPairPred = (Int, Int) => Boolean
def sizeConstraint(pred: IntPairPred, n: Int, email: Email) = pred(email.text.size, n)
~~~

Мы определили синоним для предиката, проверяющего пары целых чисел (некоторое число `n` и
размер длины письма). 

Обратите внимание на то, что в отличае от предыдущей версии, функци `sizeConstraint` 
не возвращает теперь предикат `EmailFilter`, но просто вычисляет все аргументы, переданные
в функцию и возвращает `Boolean`. Трюк состоит в том, чтобы получить нужный предикат 
с помощью частичного применения.

Но сначала, поскольку мы всерьёз решили не повторяться, давайте определим 
основные предикаты `IntPairPred`. После этого при вызове `sizeConstraint`
нам не придётся раз за разом выписывать анонимные функции:

~~~
val gt: IntPairPred = _ > _
val ge: IntPairPred = _ >= _
val lt: IntPairPred = _ < _
val le: IntPairPred = _ <= _
val eq: IntPairPred = _ == _
~~~

Наконец, всё готово для того чтобы выполнить частичное применение функции `sizeConstraint`.
Мы зафиксируем первый аргумент с одним из наших значений для `IntPairPred`:

~~~
val minimumSize: (Int, Email) => Boolean = sizeConstraint(ge, _: Int, _: Email)
val maximumSize: (Int, Email) => Boolean = sizeConstraint(le, _: Int, _: Email)
~~~

Как видно из определений, нам необходимо воспользоваться прочерком для обозначения 
пропущенных аргументов. К сожалению, нам также ппришлось явно указать типы пропущенных
аргументов. Поэтому частичное применение в Scala может быть несколько занудным.

Компилятор Scala не может вывести эти типы самостоятельно, по крайней мере не для
всех случаев -- например, для перегруженных методов компилятор не сможет понять
к какому етоду относится вызов.

С другой стороны теперь у нас есть выбор какие параметры оставить. к примеру, мы можем
оставить первый параметр и зафиксировать размер письма:

~~~
val constr20: (IntPairPred, Email) => Boolean = sizeConstraint(_: IntPairPred, 20, _: Email)
val constr30: (IntPairPred, Email) => Boolean = sizeConstraint(_: IntPairPred, 30, _: Email)
~~~

Теперь у нас есть две функции, которые принимают `IntPairPred` и `Email` и сравнивают размер
письма с числами `20` и `30`. Но то как они проводят сравнения не фиксируется. Как раз за это и отвечает
`IntPairPred`.

На этом примере видно, что несмотря на многословность, частичное применение
в Scala всё же немного более общее чем в Clojure, где мы обязаны
передавать аргументы по-порядку слева направо, но не можем пропустить 
переметр в середине.

### От методов к функциональным объектам

При частичном применении методов мы можем не связывать ни один из параметров. 
Тогда список аргументов для возвращаемого функционального объекта будет
совпадать с исходным списком аргументов для метода. Так мы можем превратить
метод в функцию, которая может быть присвоена переменной или передана в метод.

~~~
val sizeConstraintFn: (IntPairPred, Int, Email) => Boolean = sizeConstraint _
~~~

### Сщздаём EmailFilter

У нас всё ещё нет ни одной фукции, которая возвращала бы `EmailFilter`. 
Ведь `sizeConstraint`, `minimumSize` и `maximumSize` -- ни одна из этих функций
не возвращает `EmailFilter`. Все они возвращают `Boolean` как видно из сигнатур их типов.

Но наши фильтры находятся лишь в одном частичном применении от нас. 
Зафиксировав целочисленный параметр для `minimumSize` и `maximumSize` мы
можем создать новые функции типа `EmailFilter`:

~~~
val min20: EmailFilter = minimumSize(20, _: Email)
val max20: EmailFilter = maximumSize(20, _: Email)
~~~

Мы могли выполнить то же самое через частичноеприменение функции `constr20`:

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




