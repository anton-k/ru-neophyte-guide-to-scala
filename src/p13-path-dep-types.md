Part 13: Path-dependent Types
==============================================================

In last week’s article, I introduced you to the idea of type classes – a pattern that allows you to design your programs to be open for extension without giving up important information about concrete types. This week, I’m going to stick with Scala’s type system and talk about one of its features that distinguishes it from most other mainstream programming languages: Scala’s form of dependent types, in particular path-dependent types and dependent method types.

One of the most widely used arguments against static typing is that “the compiler is just in the way” and that in the end, it’s all only data, so why care about building up a complex hierarchy of types?

In the end, having static types is all about preventing bugs by allowing the über-smart compiler to humiliate you on a regular basis, making sure you’re doing the right thing before it’s too late.

Using path-dependent types is one powerful way to help the compiler prevent you from introducing bugs, as it places logic that is usually only available at runtime into types.

Sometimes, accidentally introducing path-dependent types can lead to frustation, though, especially if you have never heard of them. Hence, it’s definitely a good idea to get familiar with them, whether you decide to put them to use or not.

The problem
----------------------------------------------------

I will start by presenting a problem that path-dependent types can help us solving: In the realm of fan fiction, the most atrocious things happen – usually, the involved characters will end up making out with each other, regardless how inappropriate it is. There is even crossover fan fiction, in which two characters from different franchises are making out with each other.

However, elitist fan fiction writers look down on this. Surely there is a way to prevent such wrongdoing! Here is a first version of our domain model:

~~~
object Franchise {
   case class Character(name: String)
 }
class Franchise(name: String) {
  import Franchise.Character
  def createFanFiction(
    lovestruck: Character,
    objectOfDesire: Character): (Character, Character) = (lovestruck, objectOfDesire)
}
~~~

Characters are represented by instances of the Character case class, and the Franchise class has a method to create a new piece of fan fiction about two characters. Let’s create two franchises and some characters:

~~~
val starTrek = new Franchise("Star Trek")
val starWars = new Franchise("Star Wars")

val quark = Franchise.Character("Quark")
val jadzia = Franchise.Character("Jadzia Dax")

val luke = Franchise.Character("Luke Skywalker")
val yoda = Franchise.Character("Yoda")
~~~

Unfortunately, at the moment we are unable to prevent bad things from happening:

~~~
starTrek.createFanFiction(lovestruck = jadzia, objectOfDesire = luke)
~~~

Horrors of horrors! Someone has created a piece of fan fiction in which Jadzia Dax is making out with Luke Skywalker. Preposterous! Clearly, we should not allow this. Your first intuition might be to somehow check at runtime that two characters making out are from the same franchise. For example, we could change the model like so:

~~~
object Franchise {
  case class Character(name: String, franchise: Franchise)
}
class Franchise(name: String) {
  import Franchise.Character
  def createFanFiction(
      lovestruck: Character,
      objectOfDesire: Character): (Character, Character) = {
    require(lovestruck.franchise == objectOfDesire.franchise)
    (lovestruck, objectOfDesire)
  }
}
~~~

Now, the Character instances have a reference to their Franchise, and trying to create a fan fiction with characters from different franchises will lead to an IllegalArgumentException (feel free to try this out in a REPL).

Safer fiction with path-dependent types
---------------------------------------------------------------------------

This is pretty good, isn’t it? It’s the kind of fail-fast behaviour we have been indoctrinated with for years. However, with Scala, we can do better. There is a way to fail even faster – not at runtime, but at compile time. To achieve that, we need to encode the connection between a Character and its Franchise at the type level.

Luckily, the way Scala’s nested types work allow us to do that. In Scala, a nested type is bound to a specific instance of the outer type, not to the outer type itself. This means that if you try to use an instance of the inner type outside of the instance of the enclosing type, you will face a compile error:

~~~
class A {
  class B
  var b: Option[B] = None
}
val a1 = new A
val a2 = new A
val b1 = new a1.B
val b2 = new a2.B
a1.b = Some(b1)
a2.b = Some(b1) // does not compile
~~~

You cannot simply assign an instance of the B that is bound to a2 to the field on a1 – the one is an a2.B, the other expects an a1.B. The dot syntax represents the path to the type, going along concrete instances of other types. Hence the name, path-dependent types.

We can put these to use in order to prevent characters from different franchises making out with each other:

~~~
class Franchise(name: String) {
  case class Character(name: String)
  def createFanFictionWith(
    lovestruck: Character,
    objectOfDesire: Character): (Character, Character) = (lovestruck, objectOfDesire)
}
~~~

Now, the type Character is nested in the type Franchise, which means that it is dependent on a specific enclosing instance of the Franchise type.

Let’s create our example franchises and characters again:

~~~
val starTrek = new Franchise("Star Trek")
val starWars = new Franchise("Star Wars")

val quark = starTrek.Character("Quark")
val jadzia = starTrek.Character("Jadzia Dax")

val luke = starWars.Character("Luke Skywalker")
val yoda = starWars.Character("Yoda")
~~~

You can already see in how our Character instances are created that their types are bound to a specific franchise. Let’s see what happens if we try to put some of these characters together:

~~~
starTrek.createFanFictionWith(lovestruck = quark, objectOfDesire = jadzia)
starWars.createFanFictionWith(lovestruck = luke, objectOfDesire = yoda)
~~~

These two compile, as expected. They are tasteless, but what can we do?

Now, let’s see what happens if we try to create some fan fiction about Jadzia Dax and Luke Skywalker:

~~~
starTrek.createFanFictionWith(lovestruck = jadzia, objectOfDesire = luke)
~~~

Et voilà: The thing that should not be does not even compile! The compiler complains about a type mismatch:

~~~
found   : starWars.Character
required: starTrek.Character
               starTrek.createFanFictionWith(lovestruck = jadzia, objectOfDesire = luke)
                                                                                   ^
~~~

This technique also works if our method is not defined on the Franchise class, but in some other module. In this case, we can make use of dependent method types, where the type of one parameter depends on a previous parameter:

~~~
def createFanFiction(f: Franchise)(lovestruck: f.Character, objectOfDesire: f.Character) =
  (lovestruck, objectOfDesire)
~~~

As you can see, the type of the lovestruck and objectOfDesire parameters depends on the Franchise instance passed to the method. Note that this only works if the instance on which other types depend is in its own parameter list.


Abstract type members
---------------------------------------------------------------

Often, dependent method types are used in conjunction with abstract type members. Suppose we want to develop a hipsterrific key-value store. It will only support setting and getting the value for a key, but in a typesafe manner. Here is our oversimplified implementation:

~~~
object AwesomeDB {
  abstract class Key(name: String) {
    type Value
  }
}
import AwesomeDB.Key
class AwesomeDB {
  import collection.mutable.Map
  val data = Map.empty[Key, Any]
  def get(key: Key): Option[key.Value] = data.get(key).asInstanceOf[Option[key.Value]]
  def set(key: Key)(value: key.Value): Unit = data.update(key, value)
}
~~~

We have defined a class Key with an abstract type member Value. The methods on AwesomeDB refer to that type without ever knowing or caring about the specific manifestation of this abstract type.

We can now define some concrete keys that we want to use:

~~~
trait IntValued extends Key {
 type Value = Int
}
trait StringValued extends Key {
  type Value = String
}
object Keys {
  val foo = new Key("foo") with IntValued
  val bar = new Key("bar") with StringValued
}
~~~

Now we can set and get key/value pairs in a typesafe manner:

~~~
val dataStore = new AwesomeDB
dataStore.set(Keys.foo)(23)
val i: Option[Int] = dataStore.get(Keys.foo)
dataStore.set(Keys.foo)("23") // does not compile
~~~

Path-dependent types in practice
----------------------------------------------------------------------------------

While path-dependent types are not necessarily omnipresent in your typical Scala code, they do have a lot of practical value beyond modelling the domain of fan fiction.

One of the most widespread uses is probably seen in combination with the cake pattern, which is a technique for composing your components and managing their dependencies, relying solely on features of the language. See the excellent articles by Debasish Ghosh and Precog’s Daniel Spiewak to learn more about both the cake pattern and how it can be improved by incorporating path-dependent types.

In general, whenever you want to make sure that objects created or managed by a specific instance of another type cannot accidentally or purposely be interchanged or mixed, path-dependent types are the way to go.

Path-dependent types and dependent method types play a crucial role for attempts to encode information into types that is typically only known at runtime, for instance heterogenous lists, type-level representations of natural numbers and collections that carry their size in their type. Miles Sabin is exploring the limits of Scala’s type system in this respect in his excellent library Shapeless.


----------------------------------------------------

* <= [Глава 12: Классы типов](https://github.com/anton-k/ru-neophyte-guide-to-scala/blob/master/src/p12-type-classes.md)

* => [Глава 14: Модель акторов](https://github.com/anton-k/ru-neophyte-guide-to-scala/blob/master/src/p14-actors.md)

* [Содержание](https://github.com/anton-k/ru-neophyte-guide-to-scala#%D0%9F%D1%83%D1%82%D0%B5%D0%B2%D0%BE%D0%B4%D0%B8%D1%82%D0%B5%D0%BB%D1%8C-%D0%BD%D0%B5%D0%BE%D1%84%D0%B8%D1%82%D0%B0-%D0%BF%D0%BE-scala)