---
date: 2026-11-18
ingest: full
links:
  - https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W
---
# L06. PLINQ и Task Parallel Library

Код PLINQ восстановлен по рассказу лектора, код задач взят из архива примеров с сокращениями.

## PLINQ

PLINQ параллелит LINQ-запрос одной строкой `AsParallel()`: дальше элементы обрабатываются на пуле потоков. Это [[mtap-L05-пул-потоков-и-data-parallelism#Data parallelism и task parallelism|data parallelism]]: выигрыш есть, только если обработка элемента дорогая по CPU. ([04:24](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=264))

```csharp
var fibs = Enumerable.Range(0, 100)
    .AsParallel()                 // дальше тип ParallelQuery со своим набором методов
    .WithDegreeOfParallelism(20)
    .AsOrdered()
    .Select(Fib)                  // Fib считается на потоках пула
    .ToArray();
```

Порядок результата по умолчанию не гарантирован: PLINQ разбивает данные, обрабатывает части параллельно и сливает в одну коллекцию. Исходный порядок возвращает `AsOrdered`. ([11:34](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=694))

Настройки запроса: ([09:37](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=577))

| Метод                     | Что задаёт                                                                                                        |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `WithExecutionMode`       | `Default`: параллелить, только если PLINQ ждёт ускорения; `ForceParallelism`: всегда                              |
| `AsOrdered`               | исходный порядок результата ценой производительности                                                              |
| `WithDegreeOfParallelism` | максимум одновременных задач: защита от перегрузки, а не ускорение                                                |
| `WithMergeOptions`        | `NotBuffered`: по одному элементу; `AutoBuffered`: пачками; `FullyBuffered`: всё разом                            |
| `WithCancellation`        | токен мягкой остановки, как в [[mtap-L05-пул-потоков-и-data-parallelism#Parallel.Invoke, For, ForEach\|Parallel]] |

По лектору, PLINQ сам решает, параллелить ли, в том числе по объёму данных, и `if` на размер входа писать не нужно ([32:58](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=1978)). По документации режим `Default` смотрит на структуру запроса (вне конспекта, [docs](https://learn.microsoft.com/en-us/dotnet/api/system.linq.parallelexecutionmode)).

По лектору, `AutoBuffered` копит результаты пачками перед следующим этапом конвейера, как батч-запрос в базу ([17:49](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=1069)). По документации merge options задают только то, как итоговый результат отдаётся потоку-потребителю (вне конспекта, [docs](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/merge-options-in-plinq)).

## Ленивость против параллельности

LINQ ленив: элемент идёт дальше по конвейеру, как только готов, поэтому разные этапы могут одновременно работать с разными элементами. Методы, которые собирают всю последовательность, ломают и ленивость, и параллельность между этапами. ([19:21](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=1161))

```csharp
data.AsParallel()
    .Select(A)
    .ToArray()    // ждёт, пока A пройдут все элементы
    .Select(B);   // B не начнётся ни для одного элемента до конца A
```

Параллельность ломают: ([23:07](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=1387))
- `ToArray`, `ToDictionary`, `ToHashSet`, группировка: копят всё в коллекцию
- `AsOrdered` после неупорядоченной обработки: порядок приходится восстанавливать
- `AsSequential`: возвращает запрос в один поток

`AsSequential` ломает параллельность намеренно: тяжёлую обработку параллелят, а запросы во внешний сервис шлют по одному, чтобы не убить его сотней потоков. ([28:01](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=1681))

```csharp
data.AsParallel()
    .Select(x => x + 100)          // нагрузка на CPU: параллельно
    .AsSequential()
    .Select(x => client.Set(x))    // запросы во внешний сервис: по одному
    .ToArray();
```

`ForAll(action)` есть только у `ParallelQuery`: он выполняет действие параллельно и ничего не возвращает, поэтому завершает конвейер. ([31:05](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=1865))

## Task

**Task** - абстракция действия, которое выполняется на [[mtap-L05-пул-потоков-и-data-parallelism#Пул потоков|пуле потоков]]. Задачу можно запустить, отменить, узнать её статус и декларативно указать зависимость от других задач. ([40:35](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=2435))

**TPL** (Task Parallel Library) - библиотека C# для task parallelism, где задачи завязаны друг на друга или на IO. Пример: перевод Васи Пете и Саше, где оба зачисления ждут проверки прав, баланса и списания. ([38:40](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=2320))

```csharp
var task = new Task<string>(() => File.ReadAllText("file.txt"));
task.Start();                    // new Task сам не запускается
task.Wait();                     // блокирует поток до терминального статуса
Console.WriteLine(task.Result);  // тоже блокирует, если задача не завершена

task = Task.Run(() => File.ReadAllText("file.txt"));     // создать и запустить, самый частый способ
task = Task.Factory.StartNew(() => File.ReadAllText("file.txt"),
    TaskCreationOptions.DenyChildAttach);                 // то же, что Task.Run, но опции можно менять
```

`Wait` и `Result` блокируют поток: он ничего не делает, а потоки - ограниченный ценный ресурс. Как ждать без блокировки, разберут на следующей лекции. ([50:41](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=3041))

Исключение задачи `Wait` и `Result` бросают обёрнутым в [[mtap-L05-пул-потоков-и-data-parallelism#AggregateException|AggregateException]], даже если задача одна. ([58:11](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=3491))

TPL не спасает от [[mtap-L05-пул-потоков-и-data-parallelism#Общая очередь и thread pool starvation|thread pool starvation]]: пул не знает, сколько CPU съест задача, и мелкие задачи по-прежнему вредны. ([42:08](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=2528))

### Статусы

Промежуточные статусы нужны не для отладки, а чтобы следить за производительностью системы. ([44:27](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=2667))

| Статус                                   | Что значит                                  |
| ---------------------------------------- | ------------------------------------------- |
| `Created`, `WaitingForActivation`        | начальные                                   |
| `WaitingToRun`                           | создана, но ещё не исполняется              |
| `Running`                                | выполняет полезную работу                   |
| `WaitingForChildrenToComplete`           | своя работа закончена, ждёт дочерние задачи |
| `RanToCompletion`, `Faulted`, `Canceled` | терминальные: успех, исключение, отмена     |

- время в `WaitingToRun` - зазор между постановкой и стартом из producer-consumer, его стоит мерить
- зависшая дочерняя задача держит родителя в `WaitingForChildrenToComplete`
- `Canceled` бывает посреди работы, когда часть уже сделана, как у батч-записи на 50-м ключе

## ContinueWith

`ContinueWith` создаёт новую задачу, которая запустится на пуле после предыдущей и получит её на вход. Поток между ними может смениться: пулу всё равно, где исполнять задачу. В асинхронности это станет источником граблей. ([01:00:43](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=3643))

```csharp
var taskA = Task.Run(() => 1 + 2);
var taskB = taskA.ContinueWith(a => a.Result + 3);  // поток может быть другим
Console.WriteLine(taskB.Result);                    // 6
```

Без опций продолжение запустится, даже если предыдущая задача упала или отменена, и тогда `Result` бросит исключение. Условие задают `TaskContinuationOptions`, а продолжение с невыполненным условием получает статус `Canceled`. ([01:04:55](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=3895))

```csharp
var done = Task.FromResult(42);           // готовые задачи без запуска
var failed = Task.FromException(new InvalidOperationException());
var canceled = Task.FromCanceled(token);

canceled.ContinueWith(_ => { }, TaskContinuationOptions.OnlyOnCanceled);  // выполнится
done.ContinueWith(_ => { }, TaskContinuationOptions.OnlyOnCanceled);      // Canceled
// есть OnlyOnRanToCompletion, OnlyOnFaulted, NotOnRanToCompletion, NotOnFaulted, NotOnCanceled
```

Перевод Васи на продолжениях выглядит так, и лектор показал его как пример неудобного кода: из таких цепочек выросла асинхронность в C#. ([01:16:10](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=4570))

```csharp
var decrease = ensureAccess
    .ContinueWith(_ => CheckMoney(), TaskContinuationOptions.OnlyOnRanToCompletion)
    .ContinueWith(_ => Decrease(vasya), TaskContinuationOptions.OnlyOnRanToCompletion);
var toPetya = decrease.ContinueWith(_ => Add(petya), TaskContinuationOptions.OnlyOnRanToCompletion);
var toSasha = decrease.ContinueWith(_ => Add(sasha), TaskContinuationOptions.OnlyOnRanToCompletion);
// с появлением Task гонки никуда не делись
toPetya.Wait();
toSasha.Wait();
```

`ContinueWith` с опциями всё ещё иногда незаменим, чтобы связать задачи по хитрым условиям ([01:16:51](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=4611)). Лектор показывает разные способы связать задачи, но на практике их вытесняет асинхронность, где то же делается проще (со слов).

## Дочерние задачи

Задача, запущенная внутри другой, по умолчанию не связана с ней: родитель завершается, не дождавшись её. Такая задача утекает, и если она крутит `while (true)`, до неё уже не дотянуться. ([01:17:59](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=4679))

```csharp
var outer = Task.Factory.StartNew(() =>
{
    Task.Factory.StartNew(() =>
    {
        Thread.Sleep(1000);
        Console.WriteLine("inner task");
    }, TaskCreationOptions.AttachedToParent);   // родитель ждёт эту задачу
    Console.WriteLine("outer task");
});
outer.Wait();
Console.WriteLine("outer task completed");
// с AttachedToParent: outer task, inner task, outer task completed
// без него:           outer task, outer task completed, inner task
```

`AttachedToParent` передают только через `Task.Factory.StartNew`: у `Task.Run` нет параметра опций. С родителем из `Task.Run` прикрепление не сработало, потому что `Task.Run` равен `StartNew` с `DenyChildAttach`. ([01:21:50](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=4910))

`DenyChildAttach` у родителя запрещает прикрепление: так библиотечный метод с `AttachedToParent` не заставит ваши задачи себя ждать. ([01:23:28](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=5008))

Дождаться ребёнка через `Wait` внутри родителя тоже можно, но тогда поток пула родителя заблокирован, пока ребёнок занимает второй поток. С `AttachedToParent` поток родителя освобождается. ([01:25:06](https://kontur.ktalk.ru/recordings/LQt9elk79llIBte8Ai9W?t=5106))

На следующей лекции: зависимость задачи от нескольких задач и асинхронность.

> [!note]- Пропущено
> повтор data и task parallelism и partitioners из L05, неудавшееся демо неупорядоченного вывода PLINQ, самописные `ForAll` в проектах и его появление в .NET 10, PLINQ с Entity Framework, аналоги TPL в Java и Go, свой пул с приоритетами задач, транзакции и атомарность в базах данных, вопрос про предел вложенности задач.
