Глава 10: Не повторяемся вместе с функциями высшего порядка
===========================================================

В прошлых статьях, мы говорили о том как легко комбинировать коллекции в Scala. 
Оказывается мы можем комбинировать не только `Future`, `Try` и другие типы-колекции,
но и функции. Функции являются значениями первого класса в Scala. 

Возможность составлять из простых элементов более сложные влечёт за собой 
многократное использование кода. Переиспользование кода часто записывают
в ряд достоинств объектно ориентированного подхода, но эта черта справедлива
и для чистых функций, поскольку функции, которые не выполняют побочных эффектов
прозрачны по ссылкам.

Один из простейших способов реализации новой функции заключается в вызове других функций в её теле.
Но есть и другие способы. В этой статье мы обсудим базовые понятия функционального
программирования. Вы узнаете о том как в лучших традициях принципа DRY(Don't Repeat Youself - не повторяйтесь)
функции высшего порядка применяются для повторного использования кода. 

Функции высшего порядка
-----------------------------------------------------------

Функция высшего порядка, в отличае от функции первого порядка, имеет один из трёх видов:

1. Один из параметров функции также является функцией и она возвращает значение.

2. Она возвращает функцию, но ни один из параметров не является функцией.

3. И первый и второй пункт: функция возвращает функцию и один из параметров является функцией.

Нам уже встречалось множество примеров функций первого типа. Методы `map`, `filter`, `flatMap`
-- все они принимают функции, с помощью которых мы преобразуем или фильтруем коллекции. 
Очень асто, мы передавали анонимные функции, что иногда приводило к дублированию.

В этой статье мы сосредоточимся на последних двух типах функций высшего порядка. 
первый из них позволяет нам строить функции на основе некоторых данных, а второй
даёт возможность строить одни функции на основе других. И в том и в другом случае 
мы можем избежать дублирования кода.

Из ничего появилась функция
---------------------------------------------------------------

Может показаться, что в функции, которая создаёт другие функции нет особой необходимости.
Хотя нас больше интересует случай составление новых функций из ранее определённых, давайте
для начала посмотрим на примеры того как могут применятся функции, которые создают
другие функции.

Предположим, что мы оздаём приложение для обработки электронных писем. Пользователь 
хочет настроить фильтр блокируемых сообщений. В нашем приложении письма представлены 
таким простым `case`-классом:

~~~
case class Email(
  subject: String,
  text: String,
  sender: String,
  recipient: String)
~~~

Мы хотим филтровать сообщения на основе некоторого критерия, определённого пользователем.
В итоге у нас будет функция `Email => Boolean`, с помощью которой мы будем фильтровать письма.
Если функция вернёт истину мы принимаем письмо, в противном случае -- отбрасываем.

~~~
type EmailFilter = Email => Boolean
def newMailsForUser(mails: Seq[Email], f: EmailFilter) = mails.filter(f)
~~~

Обратите внимание на то, как мы дали специальное имя нашей функции. Мы объявили новый тип-синоним
для повышения читаемости кода.

Теперь мы можем создать методы-фабрики, создающие `EmailFilter` на основе предпочтений пользователей:

~~~
val sentByOneOf: Set[String] => EmailFilter =
  senders => email => senders.contains(email.sender)

val notSentByAnyOf: Set[String] => EmailFilter =
  senders => email => !senders.contains(email.sender)

val minimumSize: Int => EmailFilter = n => email => email.text.size >= n

val maximumSize: Int => EmailFilter = n => email => email.text.size <= n
~~~

Каждая из этих четырёх переменных возвращает `EmailFilter`. Первые две принимают `Seq[String]`,
представляющую набор адресатов-отправителей. Оставшиеся принимают целое число, указывающее на
длину содержания письма.

С помощью любой из этих функций мы можем создать новый фильтр `EmailFilter` для функции `newMailsForUser`:

~~~
val emailFilter: EmailFilter = notSentByAnyOf(Set("johndoe@example.com"))

val mails = Email(
  subject = "It's me again, your stalker friend!",
  text = "Hello my friend! How are you?",
  sender = "johndoe@example.com",
  recipient = "me@example.com") :: Nil

newMailsForUser(mails, emailFilter) // returns an empty list
~~~

Этот фильтр удалит одно сообщение, поскольку пользователь решил добавить отправителя
в чёрный список. Так с помощью методов-фабрик мы можем создавать самые разные
фильтры сообщений `EmailFilter`, согласно пользовательским предпочтениям.

Переиспользование функций
------------------------------------------------------


There are two problems with the current solution. First of all, there is quite a bit of duplication in the predicate factory functions above, when initially I told you that the composable nature of functions made it easy to stick to the DRY principle. So let’s get rid of the duplication.

To do that for the minimumSize and maximumSize, we introduce a function sizeConstraint that takes a predicate that checks if the size of the email body is okay. That size will be passed to the predicate by the sizeConstraint function:

~~~
type SizeChecker = Int => Boolean
val sizeConstraint: SizeChecker => EmailFilter = f => email => f(email.text.size)
~~~

Now we can express minimumSize and maximumSize in terms of sizeConstraint:

~~~
val minimumSize: Int => EmailFilter = n => sizeConstraint(_ >= n)
val maximumSize: Int => EmailFilter = n => sizeConstraint(_ <= n)
~~~

Function composition
--------------------------------------------------------


For the other two predicates, sentByOneOf and notSentByAnyOf, we are going to introduce a very generic higher-order function that allows us to express one of the two functions in terms of the other.

Let’s implement a function complement that takes a predicate A => Boolean and returns a new function that always returns the opposite of the given predicate:

~~~
def complement[A](predicate: A => Boolean) = (a: A) => !predicate(a)
~~~

Now, for an existing predicate p we could get the complement by calling complement(p). However, sentByAnyOf is not a predicate, but it returns one, namely an EmailFilter.

Scala functions provide two composing functions that will help us here: Given two functions f and g, f.compose(g) returns a new function that, when called, will first call g and then apply f on the result of it. Similarly, f.andThen(g) returns a new function that, when called, will apply g to the result of f.

We can put this to use to create our notSentByAnyOf predicate without code duplication:

~~~
val notSentByAnyOf = sentByOneOf andThen(g => complement(g))
~~~

What this means is that we ask for a new function that first applies the sentByOneOf function to its arguments (a Set[String]) and then applies the complement function to the EmailFilter predicate returned by the former function. Using Scala’s placeholder syntax for anonymous functions, we could write this more concisely as:

~~~
val notSentByAnyOf = sentByOneOf andThen(complement(_))
~~~

Of course, you will now have noticed that, given a complement function, you could also implement the maximumSize predicate in terms of minimumSize instead of extracting a sizeConstraint function. However, the latter is more flexible, allowing you to specify arbitrary checks on the size of the mail body.

### Composing predicates

Another problem with our email filters is that we can currently only pass a single EmailFilter to our newMailsForUser function. Certainly, our users want to configure multiple criteria. We need a way to create a composite predicate that returns true if either any, none or all of the predicates it consists of return true.

Here is one way to implement these functions:

~~~
def any[A](predicates: (A => Boolean)*): A => Boolean =
  a => predicates.exists(pred => pred(a))
def none[A](predicates: (A => Boolean)*) = complement(any(predicates: _*))
def every[A](predicates: (A => Boolean)*) = none(predicates.view.map(complement(_)): _*)
~~~

The any function returns a new predicate that, when called with an input a, checks if at least one of its predicates holds true for the value a. Our none function simply returns the complement of the predicate returned by any – if at least one predicate holds true, the condition for none is not satisfied. Finally, our every function works by checking that none of the complements to the predicates passed to it holds true.

We can now use this to create a composite EmailFilter that represents the user’s configuration:

~~~
val filter: EmailFilter = every(
    notSentByAnyOf(Set("johndoe@example.com")),
    minimumSize(100),
    maximumSize(10000)
  )
~~~

### Composing a transformation pipeline

As another example of function composition, consider our example scenario again. As a freemail provider, we want not only to allow user’s to configure their email filter, but also do some processing on emails sent by our users. These are simple functions Email => Email. Some possible transformations are the following:

~~~
val addMissingSubject = (email: Email) =>
  if (email.subject.isEmpty) email.copy(subject = "No subject")
  else email
val checkSpelling = (email: Email) =>
  email.copy(text = email.text.replaceAll("your", "you're"))
val removeInappropriateLanguage = (email: Email) =>
  email.copy(text = email.text.replaceAll("dynamic typing", "**CENSORED**"))
val addAdvertismentToFooter = (email: Email) =>
  email.copy(text = email.text + "\nThis mail sent via Super Awesome Free Mail")
~~~

Now, depending on the weather and the mood of our boss, we can configure our pipeline as required, either by multiple andThen calls, or, having the same effect, by using the chain method defined on the Function companion object:

~~~
val pipeline = Function.chain(Seq(
  addMissingSubject,
  checkSpelling,
  removeInappropriateLanguage,
  addAdvertismentToFooter))
~~~

Higher-order functions and partial functions
----------------------------------------------------------

I won’t go into detail here, but now that you know more about how you can compose or reuse functions by means of higher-order functions, you might want to have a look at partial functions again.

### Chaining partial functions

In the article on pattern matching anonymous functions, I mentioned that partial functions can be used to create a nice alternative to the chain of responsibility pattern: The orElse method defined on the PartialFunction trait allows you to chain an arbitrary number of partial functions, creating a composite partial function. The first one, however, will only pass on to the next one if it isn’t defined for the given input. Hence, you can do something like this:

~~~
val handler = fooHandler orElse barHandler orElse bazHandler
~~~

### Lifting partial functions

Also, sometimes a PartialFunction is not what you need. If you think about it, another way to represent the fact that a function is not defined for all input values is to have a standard function whose return type is an Option[A] – if the function is not defined for an input value, it will return None, otherwise a Some[A].

If that’s what you need in a certain context, given a PartialFunction named pf, you can call pf.lift to get the normal function returning an Option. If you have one of the latter and require a partial function, call Function.unlift(f).

Summary
--------------------------------------------------

In this article, we have seen the value of higher-order functions, which allow you to reuse existing functions in new, unforeseen contexts and compose them in a very flexible way. While in the examples, you didn’t save much in terms of lines of code, because the functions I showed were rather tiny, the real point is to illustrate the increase in flexibility. Also, composing and reusing functions is something that has benefits not only for small functions, but also on an architectural level.

In the next article, we will continue to examine ways to combine functions by means of partial function application and currying.





