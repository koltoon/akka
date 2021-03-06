## Акка и модель памяти Java

Основным преимуществом использования Lightbend Platform, включая Scala и Akka, является то, что он упрощает процесс 
написания параллельного программного обеспечения. В этой статье обсуждается, как Lightbend Platform и Akka, в частности,
 подходят к общей памяти в параллельных приложениях.

### Модель памяти Java
До Java 5 была определена модель памяти Java (JMM). Можно было получить всевозможные странные результаты, когда доступ 
к общей памяти осуществлялся несколькими потоками, такими как:

* поток, не видящий значения, написанные другими потоками: проблема видимости
* поток, наблюдающий «невозможное» поведение других потоков, вызванное выполнением команд в ожидаемом порядке: проблема 
переупорядочения команд.

С внедрением JSR 133 в Java 5 многие из этих проблем были решены. JMM представляет собой набор правил, основанных на 
отношении «случится раньше», которые ограничивают, когда один доступ к памяти должен происходить перед другим, и, 
наоборот, когда им разрешено не работать. Два примера этих правил:

* **правило блокировки монитора**: релиз блокировки происходит перед каждым последующим приобретением одной и той же блокировки.
* **правило изменчивой переменной**: запись изменчивой переменной `происходит до` каждого последующего чтения одной и той 
же изменчивой переменной

Хотя JMM может показаться сложным, спецификация пытается найти баланс между простотой использования и возможностью 
писать эффективные и масштабируемые параллельные структуры данных.

### Акторы и модель памяти Java
С реализацией Actors в Akka существует два способа, которыми несколько потоков могут выполнять действия в общей памяти:

* если сообщение отправляется актору (например, другим актором). В большинстве случаев сообщения являются неизменяемыми,
 но если это сообщение не является надлежащим образом сконструированным неизменным объектом, без правила `происходит до`, получатель сможет видеть частично инициализированные структуры данных;
* если актор вносит изменения во внутреннее состояние во время обработки сообщения и обращается к этому состоянию во 
время обработки другого сообщения спустя несколько секунд. Важно понимать, что с моделью актора вы не получаете никакой 
гарантии, что тот же поток будет выполнять один и тот же актор для разных сообщений.

Чтобы предотвратить видимость и переупорядочивание проблем с акторами, Akka гарантирует следующие два правила `происходит до`:

* **Правило отправки актора**: передача сообщения актору `происходит до` получения этого сообщения одним и тем же актором;
* **Правило последующей обработки актора**: обработка одного сообщения `происходит до` обработки следующего сообщения одним и
 тем же игроком.
 
>В терминах непрофессионала это означает, что изменения во внутренних полях актора видны, когда следующее сообщение 
обрабатывается этим актором. Таким образом, поля вашего актора не должны быть волотильными или эквивалентными.

Оба правила применяются только к одному экземпляру актора и недействительны, если используются разные участники.

### Фьючерсы и модель памяти Java
Завершение Будущего `происходит до` того, как выполняется вызов всех зарегистрированных обратных вызовов.

Мы рекомендуем не закрывать нефинальные поля (`final` в Java и `val` в Scala), и если вы решили закрыть не `final` поля,
 они должны быть отмечены `volatile`, чтобы текущее значение поля было видимым к обратному вызову.

Если вы закроете ссылку, вы также должны убедиться, что упомянутый экземпляр является потокобезопасным. Мы настоятельно 
рекомендуем держаться подальше от объектов, которые используют блокировку, поскольку это может привести к проблемам с 
производительностью, а в худшем случае - к взаимоблокировкам. Таковы опасности `synchronized`.

### Акторы и общее изменчивое состояние
Поскольку Akka работает на JVM, есть еще некоторые правила, которым необходимо следовать.

* Закрытие внутреннего состояния Actor и отображение его другим потокам

```scala
import akka.actor.{Actor, ActorRef}
import akka.pattern.ask
import akka.util.Timeout

import scala.concurrent.{ExecutionContextExecutor, Future}
import scala.concurrent.duration._
import scala.language.postfixOps
import scala.collection.mutable

class SharedMutableStateDocSpec {

  case class Message(msg: String)

  class EchoActor extends Actor {
    def receive = {
      case msg ⇒ sender() ! msg
    }
  }

  class CleanUpActor extends Actor {
    def receive = {
      case set: mutable.Set[_] ⇒ set.clear()
    }
  }

  class MyActor(echoActor: ActorRef, cleanUpActor: ActorRef) extends Actor {
    var state = ""
    val mySet = mutable.Set[String]()

    // это очень дорогостоящая операция
    def expensiveCalculation(actorRef: ActorRef): String =    "Meaning of life is 42"
    def expensiveCalculation(): String =      "Meaning of life is 42" 

    def receive = {
      case _ ⇒
        implicit val ec: ExecutionContextExecutor = context.dispatcher
        implicit val timeout: Timeout             = Timeout(5 seconds) // needed for `?` below

        // Пример правильного подхода
        // Полностью безопасно: "self" в порядке, чтобы закрыть и это ActorRef, который является потокобезопасным
        Future { expensiveCalculation() } foreach { self ! _ }

        // Полностью безопасно: мы закрываем фиксированное значение
        //и это ActorRef, который является потокобезопасным
        val currentSender = sender()
        Future { expensiveCalculation(currentSender) }


        //Пример неправильного подхода
        //ОЧЕНЬ ПЛОХО: общее измененное состояние приведет к
        //приложение разбивается на странные пути
        Future { state = "This will race" }
        (echoActor ? Message("With this other one"))
          .mapTo[Message]
          .foreach { received ⇒ state = received.msg
          }

        // ОЧЕНЬ ПЛОХО: shared mutable object позволяет другой актору мутировать ваше собственное состояние,
        // или хуже, вы можете получить странные условия гонки
        cleanUpActor ! mySet

        //ОЧЕНЬ ПЛОХО: «отправитель» изменяется для каждого сообщения, общая ошибка измененного состояния
        Future { expensiveCalculation(sender()) }
    }
  }
}
```
[=> далее: Надежность доставки сообщений](https://github.com/steklopod/akka/blob/akka_starter/src/main/resources/readmes/concepts/message-delivery-reliability.md)

_Если этот проект окажется полезным тебе - нажми на кнопочку **`★`** в правом верхнем углу._

[<= содержание](https://github.com/steklopod/akka/blob/akka_starter/readme.md)