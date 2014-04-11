Глава 13: Зависимые от пути типы
==============================================================

На прошлой неделе мы узнали о классах типов. Этот шаблон проектирования позволяет
нам писать расширяемые программы без потери важной информации о конкретных типах. 
В этой статье мы поговорим о возможности языка Scala, которая сильно отличает 
его от всех популярных языков программирования. В Scala мы можем воспользоваться
зависимыми типами (dependent type). В частности мы поговорим о типах завсимых от пути
(path-dependent type). 

Чаще всего языки со статической типизацией ругают за то, что "компилятор мешается под ногами",
и в конце концов всё это все волишь о данных, зачем заморачиваться над построением иерархий типов?

Статическая система типизации позволяет нам предотвращать ошибки. Умный компилятор отругает нам, 
если мы попытаемся сделать что-нибудь глупое, задолго до того, как будет слишком поздно.

Столкновение с ошиками, которые связаны с типами зависимыми от пути, могут сильно удивить Вас,
поэтому лучше заранее разобраться с этим понятием.

Проблема
----------------------------------------------------

Начнём с проблемы, которая может быть решена с помощью типов зависимых от пути. 
Обратимся к любительским сочинениям на основе научной фантастики. Там рано или поздно персонажи делают самое ужасное --
нагло целуются, не обращая внимание на время и окружающие обстоятельства. Бывает и 
такие сюжеты, в которых сходятся персонажи из разных первоисточников. 

Но элитарные писатели никогда не допустят такого. Есть способ предотвращения таких ситуаций.
Рассмотрим первую попытку:

~~~scala
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

Персонажы представлены `case`-классом `Character` и в классе первоисточника `Franchise` есть метод 
для создания новой истории двух персонажах. Давайте создадим нексколько историй:

~~~scala
val starTrek = new Franchise("Star Trek")
val starWars = new Franchise("Star Wars")

val quark = Franchise.Character("Quark")
val jadzia = Franchise.Character("Jadzia Dax")

val luke = Franchise.Character("Luke Skywalker")
val yoda = Franchise.Character("Yoda")
~~~

К сожалению, пока мы не можем избежать неприятностей:

~~~scala
starTrek.createFanFiction(lovestruck = jadzia, objectOfDesire = luke)
~~~

Какой кошмар! Кто-то создал историю, в которой Джадзия Дакс целуется с Люком Скайвокером. 
Что за нелепость! Мы должны предотвратить это. Первое, что может прийти в голову, так это
проверить на этапе выполнения, что два персонажа являются персонажами из одного первоисточника. 
К примеру мы можем изменить модель так:

~~~scala
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

У персонажей появилась ссылка на первоисточник `Franchise` и теперь при
создании сюжета мы проверяем, что персонажи пришли из одного первоисточника (Вы можете
легко в этом убдеиться в интерпретаторе).


Более надёжный способ проверки с типами зависимыми от путей
---------------------------------------------------------------------------

Этот метод не так уж плох, не так ли? Мы уже давно успели привыкнуть к нему. Но в Scala есть
способ лучше. Мы можем проверить этот условие на этапе компиляции. Для этого нам нужно закодировать
зависимость `Character` от `Franchise` на уровне типов.

К счастью, мы можем сделать это с помощью вложенныъ типов. В Scala вложенный тип связывается со значением внешнего типа,
а не с самим типом. Это означает, что если мы попытаемся воспользоваться значением внутреннего
типа за пределами значения окружающего типа, компилятор сообщит нам об ошибке:

~~~scala
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

Мы не можем просто присвоить полю переменной `a1` значение `B`, которое связанное с `a2`. Она
ожидает значения типа `a1.B`, а мы передаём `a2.B`. Синтаксис вызова через точку представляет 
собой путь к типу, по конкретному значению. Поэтому такие типы называются зависимыми от пути.

Мы можем воспользоваться этой техникой для решения исходной задачи о целующихся персонажах:

~~~scala
class Franchise(name: String) {
  case class Character(name: String)
  def createFanFictionWith(
    lovestruck: Character,
    objectOfDesire: Character): (Character, Character) = (lovestruck, objectOfDesire)
}
~~~

Теперь тип `Character` -- вложен в тип `Franchise`. Это означает, что он зависит от специфического
значения типа `Franchise`.

Давайте снова создадим наш пример с сюжетами:

~~~scala
val starTrek = new Franchise("Star Trek")
val starWars = new Franchise("Star Wars")

val quark = starTrek.Character("Quark")
val jadzia = starTrek.Character("Jadzia Dax")

val luke = starWars.Character("Luke Skywalker")
val yoda = starWars.Character("Yoda")
~~~

Из примера видно, что значения `Character` создаются так, что их типы связаны со специфическими
первоисточниками. Давайте посмотрим, что случится, если мы попытаемся совместить персонажей:

~~~scala
starTrek.createFanFictionWith(lovestruck = quark, objectOfDesire = jadzia)
starWars.createFanFictionWith(lovestruck = luke, objectOfDesire = yoda)
~~~

Эти примеры успешно скомпилируются, без неожиданностей. Безвкусица, конечно, но что поделаешь.

Теперь давайте попробуем состряпать сюжет о Джадзия Дакс и Люке Скайвокере:

~~~scala
starTrek.createFanFictionWith(lovestruck = jadzia, objectOfDesire = luke)
~~~

Вуаля! Этот пример не компилируется! Компилятор пожалуется на несоответствие типов:

~~~scala
found   : starWars.Character
required: starTrek.Character
               starTrek.createFanFictionWith(lovestruck = jadzia, objectOfDesire = luke)
                                                                                   ^
~~~

Эта техника работает, если наш метод определён не в классе `Franchise`, но в каком-нибудь дркгом модуле.
В этом случае мы можем воспользоваться типами, которые зависят от пути, сославшись на тип, который
содержится в одном из предыдущих параметров:

~~~scala
def createFanFiction(f: Franchise)(lovestruck: f.Character, objectOfDesire: f.Character) =
  (lovestruck, objectOfDesire)
~~~

Видно, что тип аргументов `lovestruck` и `objectOfDesire` зависит от значения `Franchise`, 
которое передаётся в метод первым параметром. Обратите внимание, что это работает только в том
случае, если значение, которое содержит зависимый тип находится в своём собственном списке аргументов.


Абстрактные вложенные типы
---------------------------------------------------------------

Часто зависимые типы используется совместно с абстрактными вложенными типами. Предположим,
что нам хочется создать хипстерскую базу хранения значений по ключам. Она будет поддерживать
лишь операции задания значения и чтения значения по ключу. Но делать это безопасно относительно типов.
Вот наша очень упрощённая реализация: 

~~~scala
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

Мы определили класс `Key` с абстрактным типом `Value`. Методы `AwesomeDB` пользуются
этими типами, совсем не зная о конкретной реализации абстрактного типа.

Теперь мы можемобъявить конкретные ключи:

~~~scala
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

Теперь мы можем устанавливать и читать значения с помощью
безопасных по типам методов:
~~~scala
val dataStore = new AwesomeDB
dataStore.set(Keys.foo)(23)
val i: Option[Int] = dataStore.get(Keys.foo)
dataStore.set(Keys.foo)("23") // не компилируется
~~~

Типы зависимые от пути на практике
----------------------------------------------------------------------------------

Несмотря на то, что типы зависимые от путей не встречаются на каждом шагу в типичном приложении на Scala,
они представляют очень важную возможность языка, и находят применение далеко за пределами любительской
научной фантастики. 

Один из самых распространённых случаев применения -- это шаблон пирог (cake pattern), который
позволяет регулировать зависимости в приложении средствами языка Scala. За подробностями обратитесь к статьям
[Debasish Ghosh](http://debasishg.blogspot.ie/2013/02/modular-abstractions-in-scala-with.html) и 
[Precog’s Daniel Spiewak](http://precog.com/blog/Existential-Types-FTW/). Там вы узнаете и о шаблоне пирог и какое участие в нём принимают 
типы, зависимые от пути.

В общем случае, если нам хочется завериться в том, что объекты, созданные из специфического значения
некоторого типа, не могут быть взаимозаменяемыми и принадлежат только к тому значению, из которого они
были созданы, стоит вспомнить о типах зависимыми от пути.

Типы зависимые от пути и вложенные зависимые типы играют ключевую роль в кодировании в типах
информации которая обычно доступна только на этапе выполнения программы. Примерами могут быть
гетерогенные списки, кодирование чисел с помощью системы типов и коллекции, которые хранят
в типе информацию о размере. Майлз Сабин исследует  пределы возможностей системы типов Scala
в своей замечательной библиотеке [Shapeless](https://github.com/milessabin/shapeless).


----------------------------------------------------------------------------------

* <= [Глава 12: Классы типов](https://github.com/anton-k/ru-neophyte-guide-to-scala/blob/master/src/p12-type-classes.md)

* => [Глава 14: Модель акторов](https://github.com/anton-k/ru-neophyte-guide-to-scala/blob/master/src/p14-actors.md)

* [Содержание](https://github.com/anton-k/ru-neophyte-guide-to-scala#%D0%9F%D1%83%D1%82%D0%B5%D0%B2%D0%BE%D0%B4%D0%B8%D1%82%D0%B5%D0%BB%D1%8C-%D0%BD%D0%B5%D0%BE%D1%84%D0%B8%D1%82%D0%B0-%D0%BF%D0%BE-scala)
