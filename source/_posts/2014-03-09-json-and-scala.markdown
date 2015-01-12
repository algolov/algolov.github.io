---
layout: post
title: "Json и Scala"
date: 2014-03-09 17:13:48 +1000
comments: true
categories: [scala, json, argonaut, play json, spray json]
---

В экосистеме Scala довольно много библиотек для работы с json, включая стандартную и исключая java-библиотеки. Данный пост начинает серию статей по использованию различных библиотек для работы с json в Scala. Эта серия статей не претендует на полноценный охват всех возможных библиотек или всех возможностей в библиотеках. Основная цель - понять как работать с json при помощи нескольких популярных библиотек. 
<!-- more -->
Для рассмотрения предполагается следующий список (порядок случайный):

  Статья   |   Библиотека        |   Описание
---------- | :-----------------: | ----------------------------
 [play json](http://algolov.github.io/blog/2014/03/09/json-and-scala-play-json-library) | [Play Json](http://www.playframework.com/documentation/2.2.x/ScalaJson)           | Библиотека для работы с json выделенная из популярного web framework'а - Play
           | [Spray Json](https://github.com/spray/spray-json)          | Библиотека выделенная из проекта spray (инструментарий для организации REST/HTTP слоя поверх akka).
           | [Argonaut](http://argonaut.io)            | Библиотека использующая в своей основе только функциональную парадигму, то есть pure functional by design, активно использует ScalaZ.

>**Обновление**
>К сожалению, нет времени закончить эту серию и написать все запланированные посты. Тем не менее, все запланированные библиотеки были рассмотрены в коде, который можно посмотреть на github - https://github.com/algolov/JsonSeries.git


В конце серии планируется заключительный пост со сравнением всех библиотек. В качестве сквозного примера будет использоваться список ссылок с сайта [reddit.com в json](http://www.reddit.com/r/scala/.json), конкретно [Scala subreddit](http://www.reddit.com/r/scala). В данном посте я опишу каркас, который будет использоваться при описании каждой из библиотек.

## Каркас приложения ##

Если посмотреть на [описание json объектов](https://www.blogger.com/github.com/reddit/reddit/wiki/JSON) возвращаемых JSON API reddit'a, то мы увидим, что для представления длинного содержимого используются объекты `Listing`, которые имеют три основных поля: `before`, `after` и `data`. Первые два служат для указания элемента в листинге с которого начинать разделение. А вот поле `data` как раз содержит список элементов, которые оборачивает данный листинг, в нашем примере это будут ссылки в сабреддите Scala. Так как основная цель это сравнить различные json библиотеки, а не написать свой клиент для reddit.com все модели предельно упрощены и заточены на то, чтобы показать различные сценарии работы с json. Итак, класс представляющий листинг:

```scala
case class Listing (
  id: UUID = UUID.randomUUID(),
  data: Seq[Link],
  before: Option[String],
  after: Option[String])
```

Поле id было добавлено просто для того, что бы смоделировать сценарий, когда в модели есть поля, которые не должны быть сериализованны, но должны быть заданы при десериализации. Можно представить, что мы хотели бы хранить экземпляры листингов для каких-нибудь своих загадочных целей и различать их по uuid.
Следующий класс `Link`, представляющий ссылку:

```scala
case class Link(
  title: String,
  url: URL,
  stats: Counters)
```

Оригинальный объект `Link` содержит гораздо большее количество полей, но нам вполне хватит нескольких. Последнее поле - stats, представляет собой различную количественную информацию о ссылке, выделенную в отдельный класс:

```scala
case class Counters (
  ups: Int,
  downs: Int,
  score: Int,
  num_comments: Int)
```

Итак, представленный набор классов позволит рассмотреть распространенные сценарии работы с json. Теперь перейдем к описанию рассматриваемой функциональности json библиотек. Вся функциональность выливается в четыре абстрактных метода трейта `JsonLibrary`:

```scala
trait JsonLibrary {
  type JSON
  def parseFromString (jsonStr: String): JSON
  def parseToString (json: JSON): String
  def serialize(listing: Listing): JSON
  def deserialize(json: JSON): Option[Listing]
}
```

Так как каждая библиотека для кодирования понятия JSON имеет свой конкретный тип, то в трейте присутствует абстрактное поле, представляющее этот тип (abstract type member): `type JSON`. Каждая рассматриваемая библиотека будет расширять данный трейт, конкретизируя какой тип будет использоваться в качестве представления JSON и реализовывать весь набор операций в соответствии со своим API.
Для того, что бы получить начальное строковое представление json с которым будет вестись работа, напишем вспомогательный метод, который будет считывать соответствующие данные с сайта www.reddit.com или из заранее подготовленного локального файла, расположенного в ресурсах:

```scala
def loadListing(
  count: Int = 0,
  before: Option[String] = None,
  after: Option[String] = None): String = {

  val scalaSubredditUrl = "http://www.reeddit.com/r/scala/.json"

  val resp = Try {
    (before, after) match {
      case (Some(b), _) =>
        Source.fromURL(scalaSubredditUrl + s"?count=$count&before=$b")
      case (_, Some(a)) =>
        Source.fromURL(scalaSubredditUrl + s"?count=$count&after=$a")
      case _ => Source.fromURL(scalaSubredditUrl)
    }
  } getOrElse Source.fromURL(getClass.getResource("/reddit.json"))

  resp.getLines().mkString
}
```

Параметры данного метода позволяют "пролистать" листинги, указав направление и количество просмотренных ссылок (см. [reddit api](http://www.reddit.com/dev/api)). И для того что бы удостоверится, что все работает верно - напишем несколько тестов, для этого будет использоваться замечательный тестовый фреймворк ScalaTest. 

```scala
trait UnitJsonSpec extends FlatSpec with Matchers with OptionValues with JsonLibrary {
  val jsonString = loadListing()
}
```

Трейт `UnitJsonSpec` является базовым трейтом. Он смешивает общие для всех спецификаций трейты и загружающий текст с json.

```scala
trait JsonLibraryFunctionalitySpec extends UnitJsonSpec {
  def name: String
 
  name should "parse a String representing a json, and return it as a json value and vice versa" in {
    val strJson = """{"key1":1,"key2":[1,2,3,4,5]}"""
 
    parseToString(parseFromString(strJson)) shouldBe strJson
  }
 
  it should "serialize and deserialize a json to scala classes and vice versa" in {
 
    import com.github.algolov.Listing
 
    val json = parseFromString(jsonString)
    val listing = deserialize(json).value
 
    listing shouldBe a [Listing]
 
    val listing2 = deserialize(serialize(listing)).value
 
    listing2 should have (
      'before (listing.before),
      'after (listing.after) )
 
    listing2.data.length should be (listing.data.length)
  }
}
```

Трейт `JsonLibraryFunctionalitySpec` это спецификация содержащая тесты основной функциональности библиотек. Все спецификации рассматриваемых библиотек расширяют данную спецификацию, реализуя метод name и наследуя все тесты, а так же примешивают конкретную реализацию `JsonLibrary` для переопределения абстрактных методов.

```scala
trait JsonLibraryPerfomanceSpec extends UnitJsonSpec {
  def name: String
  val numIterations = 2000
 
  def repeat[A](n: Int = 1)(f: => A) {
    if(n > 0) { f; repeat(n-1)(f) }
  }
 
  val json = parseFromString(jsonString)
  val listing = deserialize(json).value
 
  name should s"parse a json $numIterations times (iteration )" in {
    repeat(numIterations) {
      parseFromString(jsonString)
    }
  }
 
  name should s"deserialize from a json $numIterations times" in {
    repeat(numIterations) {
      deserialize(json)
    }
  }
 
  name should s"serialize to a json $numIterations times" in {
    repeat(numIterations) {
      serialize(listing)
    }
  }
}
```

`JsonLibraryPerfomanceSpec` спецификация, содержащая тесты производительности json библиотек. Данные тесты, как и тесты обзорной функциональности, не претендуют на какой-либо серьезный бэнчмаркинг и добавлены лишь для поверхностного сравнения. Так что, если интересует серьезный тест производительности json библиотек, то не стоит принимать в расчет цифры полученные в результате этих тестов.

## Исходники и запуск ##

Исходные тексты можно найти в соответствующем [репозитории на github](https://github.com/algolov/JsonSeries.git). Для запуска необходимо клонировать репозиторий, зайти в директорию JsonSeries и в командной строке выполнить команду `sbt test`. Для того, что бы запустить только тесты производительности нужно выполнить следующую команду: `sbt "test-only *PerfomanceSpec"` по этому же принципу можно указать и тесты конкретных библиотек или просмотреть только тесты функциональности. Данные команды загрузят необходимые зависимости, скомпилируют исходные файлы и запустят тесты на выполнение.