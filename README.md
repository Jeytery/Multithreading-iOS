# Многопоточность в iOS

## Так сказать база

**Синхроность** - свойство говорит о том, что таски будут выполгнятся один за другим 

--task1-taaaaaask2-taask3-------->

**Асинхроность** - свойство говорит о том, выполнение таска может оборваться (с сохранением его стейта) и начнет выполнятся другой таск. Это называется Switch Context. Таски за счет этого фейково выполняются параллельно 

--ta-TAS-TaSk-sk1-K2-3----------->

**Многопоточность** - Термин говорит о том, что таски могут выполнятся по-настоящему параллельно, на нескольких потоках. Или же даже один таск на нескольких потоках (но программист должен уметь распределять данные так, чтобы потоки не перезаписывали их в непонятной логике. Выйдет так, что не будет гарантии, что определенные данные будут в определенный момент программы, потому что какой-то поток может их перезаписать. Это называется race condition)

**Асинхронная многопоточность** - какой-то ад, таски выполнятся на многих потоках, потоки могут приостанавлиать их выолнение в пользу других и продолжать потом. 


## GCD

**Grand Central Dispatch** - это системный фреймворк эпл, написанный на Си. Эффективно удовлетворяет потребности приложения. Имеет поддержку Swift API. Поддерживается в iOS, MacOS, watchOS, tvOS.

Предоставляет доступ к очередям, которые после этого уже по своему усмотрению и возможности распределят 
задания по потокам

**Queue (очередь)** - абстракция GCD, которая принимает таски (задания, замыкания) и распределяет их на поток/и, в зависимости от своего типа 

## Serial (последовательная)
Последовательная и синхронная. Не так важно многопоточна ли она. Она лишь гарантирует для нас выполнение тасков один за другим, без асинхроности и пареллельности.

Это главная очередь и она синхроная и однопоточная. Она использует поток main для выполнения своих тасков
``` swift 
let mainQueue = DispatchQueue.main
```

## Concurrunt (параллельная)
Многопоточная очередь, чтобы выполнять таски использует много потоков. Не нашел в интернете асихронная ли она. Подозреваю что нет

Для iOS разработчика была создана специальная глобальная канкарент очередь для его задач
``` swift 
let globalQueue = DispatchQueue.global()
```
Она имеет разные QOS (quality of service) или же приоритетность 
Вот так устанавливается
``` swift 
DispatchQueue.global(qos: .background)
```
Cуществуют такие:

```.userInteractive```
задания, которые взаимодействуют с пользователем и требуют быстрой отдачи. Например быстрая обработка каких-то данных, которые надо быстро отдать на отрисовку в mainQueue

```.userInitiated ```
задания которые инициируются пользователем, но не требующие мгновенную отдачу. Могут занимать пару секунд, ответ которых пользователь ждет. 

```.unitlity```
для медленных задач. Пользователь это не вызывает, но эти процессы необходимы 

```.background```
Таски, которые не критичны по времени. Например какой-то процесс

```.defualt```
программист не предоставляет информацию для qos

## sync/async

**sync** — не имеет ничего общего с синхороностью в ее классическом понимании. Код в лямбде синка будет выполнен синхронно там где его вызывают, за счет того, что заморозиться поток в той очереди, в которой его вызывают. 

То есть синк не говорит о том, что таска закинется на очередь и будет выполнена как и все одна за другой (как сказанно в изначальном термине). 

Синк говорит лишь о том, что поток который будет обслуживать таску будет заморожен пока она не будет выполнена. Это позволяет выполнить код синхронно там где он вызывается и очередь будет лишь только обслуживать этот таск своими потоком/потоками. Если очередь однопоточная, то вся, понятное дело, она будет заморожена. 

```swift 
DispatchQueue.main.sync {
    // code
}
```

**async** — говорит нам о том, что таск будет закинут в стак очереди. И когда будут доступны потоки, то этот таск будет выполнен
Если говорить про сериал, то таск будет выполнен когда все таски до него будут выполнены. 

В канкаррент ничего не гарантированно, может сразу будет выделен поток для ее выполнения, может когда осободиться. Зависит ресурсов ОС. О том, что таск будет выполнятся со Switch Context никто не говорит (как в изначальном термине)

```swift 
DispatchQueue.main.async {
    // code
}
```
Эту специфику я не сразу понял из-за чего делал много ошибок в начале

## Проблемы мультипотока

**DEADLOCK**

В iOS oдин и больше поток не может продолжить выполнение из-за того, что ждем выполнения таски,  который может выполниться, только тогда, когда выполнитсья вторая таска, но вторая может выполниться только тогда, когда выполниться первая. В итоге они оба бесконечно ждут друг друга

Самый популярный пример
``` swift 
func printHelloWorld() {
    DispatchQueue.main.sync {
        print("Hello, World!")
    }
}
```
или если функция вызывается не в main thread-e, то 
``` swift 
DispatchQueue.main.async {
    DisptachQueue.main.sync {
        print("Hello, World!")
    }
}
```

давайте разберемся почему

весь код по стандарту вызывается в main треде (либо смотрите второй вариант)

main queue это serial queue, что означает, что она ждем пока закончится таск, после чего начнет следующий

main queue оперирует одним потоком main (что в целом не обязательно, но по большей части почти всего именно им)

дальше следует вереница из таких действий

1. мы закинули в конец очереди функцию printHelloWorld()
2. функция начала выполнятся когда поток main будет готов ее выполнять
3. в функции мы закинули в конец очереди такс написать 'Hello, World!'
 4. мы сказали сделать это синхранно, что означает заморозить поток пока эта операция не выполнится
5. из-за замороженного потока printHelloWorld() не может закончить свое выполнение
6. напечататься 'Hello, World!' так же не может, потому что из-за того что очередь serial. Нужно дождаться когда закончит свое выполнение фукнция printHelloWorld!

**LIVELOCK** 

Ситуация, в которой два потока постоянно уступают ресурсы друг другу. Как пример можно взять ситуацию когда два человека идут по коридору, чтобы пройти нужно кто-то уступил другому место.

И оба они постоянно из стороны в строну уступают другу другу так что не могут пройти

**PRIORITY INVERSION**

Cитуация, в которой более приоритетные задачи ждут выполнения менее приоритетных

**RACE CONDITION**

Cитуация, в которой ожидаемый порядок выполнения операций становится непредсказуемым, в результате чего страдает закладываемая логика. может случаться в concurrent очередях, если идет привязка к концу выполнения нескольких задач

Лично я сталкивался только в дедлоками по примеру, который я описывал. Что нужно делать, чтобы вызвать остальные проблемы - я не знаю. Подозреваю, что они могут быть, если использовать не GCD или NSQO, а создавать треды и менеджить их самому ручками

## Тулзы GCD

**DispatchGroup**
Можно закинуть много тасков и когда все они выполнятся, то группа об этом уведомит

**DispatchWorkItem**
Можно использовать замыкание, но DWI предоставляет нам удобные методы отслеживание стейта. 

**DispatchBarrier**
механизм синхронизации задач в очереди. Для того, чтобы добавить барьер, необходимо передать соответствующий флаг в метода async.

Когда мы добавляем барьер в параллельную очередь, она откладывает выполнение задачи, помеченной барьером (и все остальные, которые поступят в очередь во время выполнения такой задачи), до тех пор, пока все предыдущие задачи не будут выполнены. После того, как все предыдущие задачи будут выполнены, очередь выполнит задачу, помеченную барьером самостоятельно. Как только задача с барьером будет выполнена, очередь вернется к своему нормальному режиму работы.

## Про NSOperationQueue
NSOpertaionQueue - это обертка на GCD, написанная на Objective-C

Tак как NSOQ это обертка над GCD, то перформенс из-за нее будет немного меньше

Возможности и особености:
- Dependencies - операция не начнет свое выполнение поĸа операция от которой она зависит не закончит свою работу.
- Easy cancelation (Cancel all) - Отмена операций (например пользователь перешел на ĸонтроллер и мы начали загрузĸу больших данных, но пользователь ушел с этого ĸонтроллера и мы можем отменить операцию).
- запускает операции на глобальной, параллельной очереди.

NSOQ  для каких-то сложных манипуляций с задачами, для всего другого - GCD

## Решение сложных задач по мультипотоку 

Вам могут давать задачи по многопотоку, но по большому счету они будут сводиться к большой вложености в сириал очереди. 

Потому что если очередь будет не сириал, таски будут выполнятся в произвольном порядке 

Пример 1
``` swift 
func task() {
    print("A")
    DispatchQueue.main.async {
        print("B")
    }
    print("C")
}
```
Ответ: A C B

```Предполагается что таск вызывает в main thread-e```

Механизм такой: 
1. функция task закидывает в main thread
2. заходим в нее. Печатается 'A'
3. закидываем таску напечатать B в main queue
4. печатаем 'С'
5. функция завершает работу
6. main queue начинает работу над следующим заданием напечатать 'B'

Дальше в огрничивается кол-вом вложености

Пример 2 
``` swift 
func task() {
    print("A")
    DispatchQueue.global().sync {
        print("B")
    }
    print("C")
}
```
Ответ: A B C 

Потому что вызыв идет через синк, пока не выполнится print("B"), продолжать выполнения кода не будет. 

Будьте внимательны и обращайте внимание на 
``` swift 
DispatchQueue.main.sync {
    // code
}
```
Если это будут вызывать не в глобале, то оно выдаст дедлок

## Ссылки 

https://www.youtube.com/watch?v=GVXyrLB1tbk&list=PLNSmyatBJig7GmFpPEr9oBiFSBai7V3dC&index=3&t=1384s

https://www.youtube.com/watch?v=uEeFqIUXJcE&t=1491s

https://docs.google.com/document/d/1A0ROqvFwEL2UYfJIDkBI9PHeUPp5oiwHd_53OrI7SHs/edit

https://habr.com/ru/flows/admin/

https://biysk-tv.ru/raznoe-2/chto-takoe-asinxronnyj-potok-asinxronnoe-programmirovanie-koncepciya-realizaciya-primery.html

v2
