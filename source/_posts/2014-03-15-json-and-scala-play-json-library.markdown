---
layout: post
title: "Json и Scala: Play Json Library"
date: 2014-03-09 17:13:48 +1000
comments: true
categories: [scala, json, play json]
---

{% img /images/json-and-scala/play_json_pic.png 350 350 %}

Данный пост входит в [серию статей](http://algolov.github.io/blog/2014/03/09/json-and-scala/) на тему работы с JSON в Scala и в нем будет рассмотрена работа с библиотекой Play Json ([github](https://github.com/playframework/playframework/tree/master/framework/src/play-json), [документация](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.json.package)). Код основного примера и каркаса приложения можно [посмотреть в репозитории на github](https://github.com/algolov/JsonSeries.git). 
<!-- more -->
Данная библиотека была выделена из веб-фреймворка Play, который входит в Typesafe Reactive Platform и для парсинга JSON строк использует java-библиотеку [Jackson](http://jackson.codehaus.org/). Для того, что бы начать работать с библиотекой play json необходимо указать ее в качестве зависимости в вашем файле build.sbt:

```scala build.sbt https://github.com/algolov/JsonSeries/blob/master/build.sbt#L14
libraryDependencies ++= Seq(
  ...
  "com.typesafe.play" %% "play-json" % "2.2.1"
  ...
  )
```

И написать 2 строчки импорта:

```scala PlayJson.scala https://github.com/algolov/JsonSeries/blob/master/src/main/scala/com/github/algolov/libs/PlayJson.scala#L3
import play.api.libs.json._
import play.api.libs.functional.syntax._
```

Рассматривать возможности данной библиотеки я буду на основе примера описанного в [заглавном посте](http://algolov.github.io/blog/2014/03/09/json-and-scala/) серии, с которым необходимо ознакомится для полного понимания сути происходящего.
Поэтому для начала, рассмотрим то, что предлагает нам библиотека Play Json, а затем как это использовать в рамках нашего примера.

## Play Json: Основные методы ##

Для представления типов данных JSON в пакете `play.api.libs.json` существуют следующие типы данных:

```scala JsValue.scala https://github.com/playframework/playframework/blob/2.2.x/framework/src/play-json/src/main/scala/play/api/libs/json/JsValue.scala#L89
/** Represent a Json null value. */
case object JsNull extends JsValue {...}
 
/** Represent a Json boolean value.*/
case class JsBoolean(value: Boolean) extends JsValue
 
/** Represent a Json number value. */
case class JsNumber(value: BigDecimal) extends JsValue
 
/** Represent a Json string value. */
case class JsString(value: String) extends JsValue
 
/** Represent a Json array value. */
case class JsArray(value: Seq[JsValue] = List()) extends JsValue {...}
 
/** Represent a Json object value.*/
case class JsObject(fields: Seq[(String, JsValue)]) extends JsValue {...}
```

Как видно из листинга, все эти типы данных расширяют класс `JsValue`, который является супертипом для всех других JSON типов. Именно этот тип данных кодирует обобщенное понятие JSON, поэтому им мы и конкретизируем абстрактный член типа в трейте `JsonLibrary`.
Чтобы реализовать основную функциональность нашего трейта, посмотрим что нам предлагает содержащий статические методы [`объект Json`](https://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.json.Json$):

```scala Json.scala https://github.com/playframework/playframework/blob/2.2.x/framework/src/play-json/src/main/scala/play/api/libs/json/Json.scala#L16
/** Parse a String representing a json, and return it as a JsValue. */
 def parse(input: String): JsValue = ...
```

Итак, для того чтобы разобрать строку представляющую json, необходимо просто передать ее в качестве параметра методу `parse`, обратно мы получим значение типа `JsValue`.

```scala Json.scala https://github.com/playframework/playframework/blob/2.2.x/framework/src/play-json/src/main/scala/play/api/libs/json/Json.scala#L48
/** Convert a JsValue to its string representation. */
 def stringify(json: JsValue): String = ...
```

Для того, чтобы конвертировать `JsValue` обратно в строковое представление, необходимо передать значение этого типа в метод `stringify`.

С оставшимися двумя методами для сериализации/десериализации все немного сложнее:

```scala Json.scala https://github.com/playframework/playframework/blob/2.2.x/framework/src/play-json/src/main/scala/play/api/libs/json/Json.scala#L83
/** Provided a Reads implicit for its type is available, convert any object into a JsValue. */
 def toJson[T](o: T)(implicit tjs: Writes[T]): JsValue = ...
```

Для того, что бы конвертировать значение `JsValue` в класс нашей модели (десериализовать), мы передаем это значение в метод `fromJson` и на выходе получаем значение типа `JsResult[T]`, где `T` это тип модели. Конвертировать значение типа `JsValue` в другой тип можно так же воспользовавшись методом `validate` трейта `JsValue`, который так же возвращает значение типа `JsResult[T]`:

```scala JsValue.scala https://github.com/playframework/playframework/blob/2.2.x/framework/src/play-json/src/main/scala/play/api/libs/json/JsValue.scala#L70
def validate[T](implicit rds: Reads[T]): JsResult[T]
```

Так что же представляет из себя тип `JsResult`? `JsResult[T]` это монадический тип, который может быть либо `JsSuccess[T]` содержащий результат конвертации, в том случае если конвертация была удачной, либо `JsError[T]` в обратном случае и содержать список всех ошибок встреченных при конвертировании. И так как это монадический тип, он содержит соответствующие методы (flatMap, map, ...). Вкратце, в случае успешного конвертирования `JsResult` будет содержать экземпляр класса нашей модели, в обратном случае набор ошибок с указанием на каком этапе конвертирования эти ошибки встретились. Мы еще рассмотрим `JsResult` более подробно позже, пока же достаточно знать, что мы можем конвертировать значение это типа в значение типа `Option[T]` при помощи метода `asOpt`.

```scala Json.scala https://github.com/playframework/playframework/blob/2.2.x/framework/src/play-json/src/main/scala/play/api/libs/json/Json.scala#L90
/** Provided a Writes implicit for that type is available, convert a JsValue to any type. */
def fromJson[T](json: JsValue)(implicit fjs: Reads[T]): JsResult[T] = ...
```

Для сериализации класса в JSON передаем экземпляр класса в качестве параметра методу `toJson` и на выходе получаем значение `JsValue`.
Вся загвоздка с методами сериализации/десериализации в том, что у них есть дополнительный неявный список параметров, в котором передаются экземпляры классов `Reads[T]` и `Writes[T]` соответственно. Именно эти классы знают как правильно конвертировать типы Scala в `JsValue` и обратно. В Play json уже содержатся неявные (де)сериализаторы для основных типов данных Scala (`DefaultWrites` и `DefaultReads`). Но про то, как конвертировать экземпляры наших классов Play Json ничего не известно. Поэтому мы должны написать соответствующие неявные (де)сериализаторы самостоятельно и обеспечить их присутствие в области видимости там где они потребуются. Прежде чем это сделать, давайте посмотрим на код трейта `PlayJson` реализующий интерфейс `JsonLibrary`:

```scala PlayJson.scala https://github.com/algolov/JsonSeries/blob/master/src/main/scala/com/github/algolov/libs/PlayJson.scala#L9
trait PlayJson extends JsonLibrary {
  import PlayJson._
 
  type JSON = JsValue
 
  override def parseFromString(jsonStr: String) = Json.parse(jsonStr)
  override def parseToString(json: JsValue)     = Json.stringify(json)
  override def serialize(listing: Listing)      = Json.toJson[Listing](listing)
  override def deserialize(json: JsValue)       = Json.fromJson[Listing](json).asOpt
}
```

В этой реализации нет ничего особенного, мы просто используем методы описанные выше. Единственная неоговоренная строка - `import PlayJson._`. Она производит импорт содержимого объекта-компаньена `PlayJson`. В нем мы определим все необходимые неявные значения для конвертирования экземпляров наших кейс классов, данным импортом мы вводим их в область видимости, что бы они могли быть подхвачены методами нуждающимися в них. Что же содержит объект `PlayJson`? На самом деле совсем немного строк кода. Но для того, что бы написать их придется изучить немного теории.

## Play Json: Reads, Writes, Format и JsPath ##

Итак мы выяснили, что для того, чтобы конвертировать JsValue в другой тип Scala и обратно нам необходимо предоставить в область видимости соответствующие значения `Reads[T]` и `Writes[T]` где `T` класс модели. Как мы увидим позже, эти конвертеры можно объединить. Но начнем рассматривать все по порядку, а для этого нам нужно познакомиться еще с одним парнем - `JsPath`.

### JsPath ###

[JsPath](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.json.JsPath) это набор узлов которые надо обойти в структуре `JsValue`, чтобы получить значение. Попросту говоря, `JsPath` представляет собой путь до конкретного значения в json объекте, это практически тоже самое что и `XPath` для XML. Давайте посмотрим на определение `JsPath`:

```scala JsPath.scala https://github.com/playframework/playframework/blob/2.2.x/framework/src/play-json/src/main/scala/play/api/libs/json/JsPath.scala#L164
case class JsPath(path: List[PathNode] = List()) {
  def \(child: String) = JsPath(path :+ KeyPathNode(child))
  def \(child: Symbol) = JsPath(path :+ KeyPathNode(child.name))
 
  def \\(child: String) = JsPath(path :+ RecursiveSearch(child))
  def \\(child: Symbol) = JsPath(path :+ RecursiveSearch(child.name))
 
  def apply(idx: Int): JsPath = JsPath(path :+ IdxPathNode(idx))
 
  def apply(json: JsValue): List[JsValue] = path.foldLeft(List(json))((s, p) => s.flatMap(p.apply))
 
  ...
```

Как видно из листинга `JsPath` это список узлов пути `List[PathNode]` и набор операций над этим списком. Методы `\` и `\\` имеют две версии. Это нужно для того, что бы можно было сформировать путь используя как строки (`JsPath \ "key1"`), так и символы (`JsPath \ 'key1`). Оба этих метода и первый метод `apply` формируют новый путь добавляя узлы соответствующих типов (они все расширяют `PathNode`) к исходному пути. Вторая версия `apply` применяет значение типа `JsValue` к сформированному пути, т.е. берем первый узел в пути и из переданного json-значения выбираем все дочерние ключи соответствующие этому узлу. Получается список значений `JsValue` (отобранные ключи). Затем берется следующий узел и из полученного списка выбирается все дочерние ключи, соответствующие новому узлу и так далее. Когда узлы в пути закончатся, то итоговый список значений `JsValue` как раз и будет результатом. Давайте посмотрим на примере и сформируем такой путь:

```scala
scala> val path = JsPath \ "key1" \ "key2"
path: play.api.libs.json.JsPath = /key1/key2
```

А теперь попробуем применить к этому пути различные json объекты:

```scala
scala> val json1 = Json.obj("key1" -> Json.obj("key2" -> "value"))
json1: play.api.libs.json.JsObject = {"key1":{"key2":"value"}}
 
scala> val json2 = Json.obj("key1" -> Json.obj("key2" -> Json.arr(1,2,3,4,5)))
json2: play.api.libs.json.JsObject = {"key1":{"key2":[1,2,3,4,5]}}
 
scala> path(json1)
res0: List[play.api.libs.json.JsValue] = List("value")
 
scala> path(json2)
res1: List[play.api.libs.json.JsValue] = List([1,2,3,4,5])
 
scala> path(2)(json2)
res6: List[play.api.libs.json.JsValue] = List(3)
```

Итак, для того что бы сформировать путь, нам надо начать с объекта `JsPath`, который представляет корневой элемент пути и продолжить добавляя к нему соответствующие узлы, при помощи описанных выше методов. Для большего удобства и лучшего визуального выделения в объекте пакета json для объекта `JsPath` определен алиас: `__` так что путь из предыдущего примера можно переписать так:

```scala
val path = __ \ "key1" \ "key2"
```

Теперь, когда мы научились формировать путь, можно перейти к написанию непосредственно конвертеров.
Reads

Конвертеры `Reads[T]` используются для десериализации из `JsValue` в какой-нибудь другой тип `T`, например в класс модели `Listing`. `Reads[T]` можно комбинировать и вкладывать друг в друга, что бы получить более сложные конвертеры `Reads[T]`. Например для того, что бы получить десериализатор кейс класса `Counters`, нам необходимо объединить четыре десериализатора из `JsValue` в Int. А для того, что бы написать десериализатор для класса Link нам придется вложить десериализатор класса `Counters`. Давайте, напишем десериализатор для класса `Counters`. Так как все поля этого класса имеют стандартный тип Scala, это будет довольно просто и библиотека Play Json предлагает несколько способов сделать задуманное. Во первых, мы можем воспользоваться `JsPath` и его методами `read` и `readNullable`:

```scala JsPath.scala https://github.com/playframework/playframework/blob/2.2.x/framework/src/play-json/src/main/scala/play/api/libs/json/JsPath.scala#L256
def read[T](implicit r: Reads[T]): Reads[T]
def readNullable[T](implicit r: Reads[T]): Reads[Option[T]]
```

Данные методы принимают в качестве параметра неявный конвертер типа `Reads[T]` и применяют его к значению извлеченному по указанному пути. Метод `readNullable` полезен в случае если значение по указанному пути равно `null` или не найдено, в таком случае он вернет `None`. Например следующий код:

```scala
scala> val score = (__ \ "data" \ "before").readNullable[String]
score: play.api.libs.json.Reads[Option[String]] = play.api.libs.json.Reads$$anon$8@707d7290
```

Применит к значению ключа before неявный конвертер в тип String, который предоставляется Play Json:

```scala
scala> val jStr = Source.fromURL(getClass.getResource("/reddit.json")).getLines().mkString
jStr: String = ...
 
scala> val json = Json.parse(jStr)
json: play.api.libs.json.JsValue = ...
 
scala> Json.fromJson(json)(score)
res8: play.api.libs.json.JsResult[Option[String]] = JsSuccess(Some(t3_1ygz6x),/data/before)
```

Итак, мы можем извлекать и конвертировать отдельные значения. Для того, что бы преобразовать эти отдельные значения в более сложный тип, например класс `Counters` необходимо скомбинировать конвертеры при помощи операции `and` или `~` (по сути `and` просто вызывает `~`):

```scala
scala> :paste
val countersCombReads =
(__ \ 'ups).read[Int] ~
(__ \ 'downs).read[Int] ~
(__ \ 'score).read[Int] ~
(__ \ 'num_comments).read[Int]
 
// Exiting paste mode, now interpreting.
 
countersCombReads: FunctionalBuilder[Reads]#CanBuild4[Int,Int,Int,Int] = ...
```

Как видно из листинга результатом такого комбинирования является экземпляр класса `FunctionalBuilder[Reads]#CanBuild4[Int,Int,Int,Int]`, не будем останавливаться на нем подробно. Можно воспринимать этот результат как промежуточный при построении более сложных конвертеров `Reads`. Главное знать, что таких типов `FunctionalBuilder[Reads]#CanBuildX` существует вплоть до X=22, то есть мы можем скомбинировать до 22 конвертеров `Reads` (известное ограничение в Scala) и если значению такого типа передать в его метод apply функцию формирования из отдельных значений экземпляр модели (метод `apply` у кейс класса), то мы получим составной конвертер для нашей модели:

```scala
scala> val contersReads = countersCombReads.apply(Counters.apply _)
contersReads: Reads[Counters] = play.api.libs.json.Reads$$anon$8@6e266f8f
```

Более кратко все это можно записать следующим образом:

```scala
scala> implicit val countersReads = (
     | (__ \ 'ups).read[Int] ~
     | (__ \ 'downs).read[Int] ~
     | (__ \ 'score).read[Int] ~
     | (__ \ 'num_comments).read[Int]) (Counters.apply _)
countersReads: play.api.libs.json.Reads[Counters] = play.api.libs.json.Reads$$anon$8@39e24635
```

Понимая принцип создания сложных конвертеров из комбинации простых, последний листинг выглядит довольно простым. Однако библиотека Play Json предоставляет еще более простой способ. Создать десериализатор для вашей модели можно так же при помощи удобного метода reads объекта Json. Данный метод использует макросы введенные в Scala 2.10:

```
scala> Json.reads[Counters]
res25: play.api.libs.json.Reads[Test.Counters] = play.api.libs.json.Reads$$anon$8@26fbe9c1
```

Вот и все. Одна строка. Но у этого метода есть ряд ограничений:
- нельзя перегружать метод `apply` у кейс класса в объекте компаньене, потому что макрос не сможет выбрать между несколькими методами `apply`
- функции `apply` (и `unaply` в случае `write`) должны иметь соответствующие входные/выходные типы. У кейс классов это предоставляется автоматически, а для трейтов необходимо будет написать соответствующие методы `apply` и `unaply`. Обратите внимание как обрабатывается модель `Listing`, ниже
- Json макрос умеет обрабатывать следующие обобщенные типы `Option`, `Seq`, `List`, `Set` и `Map[String, _]`. В случае остальных придется отказаться от использования макросов.

Наименование полей данных в модели должно соответствовать наименованиям полей в json объекте.
Давайте теперь напишем десериализаторы для остальных моделей, начнем с модели для ссылок:

```scala
implicit val linkReads: Reads[Link] = (
  (__ \ "data" \ "title").read[String] ~
  (__ \ "data" \ "url").read[String].map[URL](new URL(_)) ~
  (__ \ "data").read[Counters]
)(Link.apply _)
```

Для поля `title` мы воспользовались неявным конвертером предоставляемым библиотекой. А для конвертирования поля `url`, я воспользовался удобным методом map и преобразовал конвертер строки в конвертер класса `URL`. Поля содержащиеся в классе `Counters`, иерархически находятся на одном уровне с полями `title` и `url`, поэтому конвертеру мы указываем просто путь `(__ \ "data")`, и так как ранее мы уже написали неявный конвертер для типа `Counters` никаких преобразований больше не требуется. Хотя, отдельный неявный конвертер для `Counters` можно было и не писать, а просто передать явно в качестве параметра:

```scala
   ...
(__ \ "data").read[Counters](Json.reads[Counters])
   ...
```

Теперь настала очередь десериализатора для последней модели:

```scala
implicit val listingFormat: Reads[Listing] = (
  (__ \ "data" \ "children").read[Seq[Link]] ~
  (__ \ "data" \ "before").readNullable[String] ~
  (__ \ "data" \ "after").readNullable[String]
  )(Listing(UUID.randomUUID(), _, _, _))
```

Здесь стоит отметить два момента. Во-первых, конвертер для поля `children`. Так как значением этого поля является массив объектов, соответствующих модели `Link` в нашем представлении, мы конвертируем его в коллекцию ссылок `Seq[Link]`. Это работает так как в play json уже есть неявный сериализатор для коллекций. Во-вторых, это способ формирования экземпляра нашей модели. Из-за того, что модель `Listing` имеет поле `id`, которое отсутствует в json, но должно быть задано при создании экземпляра, мы не можем использовать функцию `apply` модели. Вместо этого мы передаем лямбда выражение, формирующее функцию принимающую три параметра и возвращаю экземпляр класса `Lisitng`. Более развернуто это можно записать следующим образом:

```scala
(links: Seq[Link], before: Option[String], after: Option[String]) => Listing(UUID.randomUUID(), links, before, after)
```

Итак, с десериализаторами мы закончили переходим к написанию сериализаторов.

### Writes ###

С сериализаторами все будет немного проще. Дело в том, что реализация сериализатора практически не отличается от десериализатора. Начнем с модели `Counters`. Когда мы писали для нее десериализатор, то в конце концов воспользовались методом `reads[T]` объекта Json, основанным на макросах. Как вы наверно догадались, существует такой же метод и для сериализатора:

```scala
scala> Json.writes[Counters]
res10: play.api.libs.json.OWrites[Counters] = play.api.libs.json.OWrites$$anon$2@65b117cd
```

Не будем реализовывать его в качестве отдельного неявного значения, а просто передадим его в качестве параметра при реализации сериализатора для модели `Link`:

```scala
implicit val linkWrites: Writes[Link] = (
  (__ \ "data" \ "title").write[String] ~
  (__ \ "data" \ "url").write[URL](new Writes[URL]{ def writes(o: URL) = JsString(o.toString) }) ~
  (__ \ "data").write[Counters](Json.writes[Counters])
)(unlift(Link.unapply))
```

Итак, перечислим основные отличия. Для поля url мы создаем новый конвертер `Writes[URL]` и реализуем логику конвертирования в методе writes, так как к экземпляру класса `Write[T]` метод map не применим. Для того, что бы сформировать из экземпляра нашей модели, несколько значений (те члены класса которые необходимо сериализовать), мы используем функцию unapply, но так как результат этой функции будет `Option[(String, URL, Counters)]` вместо (`String`, `URL`, `Counters`) мы трансформируем ее при помощи функции `unlift`.
И в заключении сериализатор для модели `Listing`:

```scala
implicit val listingWrites: Writes[Listing] = (
  (__ \ "data" \ "children").write[Seq[Link]] ~
  (__ \ "data" \ "before").writeNullable[String] ~
  (__ \ "data" \ "after").writeNullable[String]
)(l => (l.data, l.before, l.after))
```

Единственное отличие здесь это то, что вместо метода `unapply` мы передаем лямбду возвращающую только три поля и игнорирующую поле `id`, которое не требует сериализации.
И вот наконец-то мы разобрались с сериализаторами и десериализаторами и казалось бы на этом можно ставить точку. Однако Play Json предоставляет возможность объединить написание сериализатора и десериализатора в одном классе.

### Format ###

`Format[T]` это просто смесь из конвертеров `Reads[T]` и `Writes[T]`. Для того, что бы создать `Format[T]` можно воспользоваться, например, методом на основе макросов:

```scala
scala> Json.format[Counters]
res0: play.api.libs.json.OFormat[Counters] = play.api.libs.json.OFormat$$anon$1@4c34c60e
```

Или можно воспользоваться уже имеющимися конверторами Reads и Writes:

```scala
val linkFormat: Format[Link] = Format(linkReads, linkWrites)
```

Также можно написать конвертеры с нуля, воспользовавшись комбинаторами:

```scala PlayJson.scala https://github.com/algolov/JsonSeries/blob/master/src/main/scala/com/github/algolov/libs/PlayJson.scala#L22
implicit val linkFormat: Format[Link] = (
    (__ \ "data" \ "title").format[String] ~
    (__ \ "data" \ "url").format[String].inmap[URL](new URL(_), _.toString) ~
    (__ \ "data").format[Counters](Json.format[Counters])
  )(Link.apply, unlift(Link.unapply))
```

Для поля `title` используется конвертер по умолчанию. Для поля `url` мы преобразовываем стандартный конвертер `Format[String]` в конвертер `Format[URL]` воспользовавшись методом `inmap` и передав в него функции для создания `URL` из строки и обратно. И так как `Format[Link]` объединяет в себе сериализатор и десериализатор, результатом объединения путей будет тип `FunctionalBuilder[OFormat]#CanBuild3[String,URL,Counters]`, в функцию apply которого необходимо передать две функции для построения объекта модели из отдельных значений его полей и обратно. В создании `Format[Listing]` тоже нет ничего неожиданного:

```scala PlayJson.scala https://github.com/algolov/JsonSeries/blob/master/src/main/scala/com/github/algolov/libs/PlayJson.scala#L28
implicit val listingFormat: Format[Listing] = (
    (__ \ "data" \ "children").format[Seq[Link]] ~
    (__ \ "data" \ "before").formatNullable[String] ~
    (__ \ "data" \ "after").formatNullable[String]
    )(Listing(UUID.randomUUID(), _, _, _), l => (l.data, l.before, l.after))
```

Итак, разобравшись с теорией, вернемся к объекту PlayJson нашего приложения. Вот его реализация:

```scala PlayJson.scala https://github.com/algolov/JsonSeries/blob/master/src/main/scala/com/github/algolov/libs/PlayJson.scala#L20
object PlayJson {
 
  implicit val linkFormat: Format[Link] = (
    (__ \ "data" \ "title").format[String] ~
    (__ \ "data" \ "url").format[String].inmap[URL](new URL(_), _.toString) ~
    (__ \ "data").format[Counters](Json.format[Counters])
  )(Link.apply, unlift(Link.unapply))
 
  implicit val listingFormat: Format[Listing] = (
    (__ \ "data" \ "children").format[Seq[Link]] ~
    (__ \ "data" \ "before").formatNullable[String] ~
    (__ \ "data" \ "after").formatNullable[String]
    )(Listing(UUID.randomUUID(), _, _, _), l => (l.data, l.before, l.after)) 
}
```

Обратите внимание на порядок объявления неявных значений. Так как `linkFormat` используется при создании `listingFormat`, в таком виде он обязательно должен предшествовать созданию `listingFormat`. Иначе, при выполнении тестов, вы получите не очень информативное исключение времени исполнения - `java.lang.NullPointerException`

### Тесты ###

После того как мы разобрались как сериализовывать/десериализовывать данные при помощи библиотеки Play Json. Давайте реализуем и запустим тесты. Для этого просто создадим две спецификации, расширив соответствующие спецификации для тестов функциональности и тестов производительности:

```scala PlayJsonFunctionalitySpec.scala https://github.com/algolov/JsonSeries/blob/master/src/test/scala/com/github/algolov/libs/PlayJsonFunctionalitySpec.scala#L3
class PlayJsonFunctionalitySpec extends JsonLibraryFunctionalitySpec with PlayJson {
  override def name = "Play Json"
}
```

и

```scala PlayJsonPerfomanceSpec.scala https://github.com/algolov/JsonSeries/blob/master/src/test/scala/com/github/algolov/libs/PlayJsonPerfomanceSpec.scala
class PlayJsonPerfomanceSpec extends JsonLibraryPerfomanceSpec with PlayJson {
  def name = "Play Json"
}
```

Запустим и посмотрим на результаты:

{% img /images/json-and-scala/play_json_tests.png %}