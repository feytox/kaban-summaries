---
date: 2026-11-25
ingest: full
links:
  - https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg
---
# L07. Ожидание нескольких задач, асинхронность, async/await

Код из архива примеров с сокращениями; таймаут восстановлен по рассказу, `SendAsync`, `items` и `ProcessAsync` мои.

## Ожидание нескольких задач

`Task.WhenAll` и `Task.WhenAny` возвращают задачу, которая завершится, когда завершатся все задачи массива или хотя бы одна. Это готовая замена самодельному ожиданию через продолжения с [[mtap-L06-plinq-и-tpl#Дочерние задачи|AttachedToParent]]. ([13:13](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=793))

```csharp
Task whenAll = Task.WhenAll(tasks);         // без результата; для задач Task<T> вернёт T[]
Task<Task> whenAny = Task.WhenAny(tasks);   // результат - первая завершившаяся задача

Task.WaitAll(tasks);   // блокирующие аналоги: WhenAll + Wait
Task.WaitAny(tasks);   // WhenAny + Wait
```

Классический пример `WhenAny` - таймаут запроса в другой сервис: ([14:49](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=889))

```csharp
var request = SendAsync(token);                     // token отменится по таймауту клиента
var budget = Task.Delay(TimeSpan.FromSeconds(10));  // задача, которая завершится через 10 с
var completed = Task.WhenAny(request, budget);
completed.Wait();                                   // пока блокирующее ожидание
if (!request.IsCompleted && budget.IsCompletedSuccessfully)
    Console.WriteLine("timeout!");                  // сервис ответил бы клиенту 408
```

`IsCompleted` истинно в любом терминальном статусе, в том числе после исключения, поэтому успех проверяют через `IsCompletedSuccessfully`. ([17:06](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=1026))

Оставшиеся задачи `WhenAny` не отменяет: для этого в запрос передают [[mtap-L05-пул-потоков-и-data-parallelism#Parallel.Invoke, For, ForEach|CancellationToken]], например созданный из таймаута клиентского запроса. ([18:27](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=1107))

## Синхронность и асинхронность

**Синхронное исполнение** - задачи выполняются по порядку, одна за другой. Синхронность говорит только о порядке, а не об одновременности: синхронно работают и в одном потоке, и в нескольких. ([24:45](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=1485))

Проблема синхронности - блокировки: шаг, ждущий IO, держит поток без дела. ([28:39](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=1719)) Если так разом заблокируются все воркеры, ресурсы есть, но заняты ожиданием: это [[mtap-L05-пул-потоков-и-data-parallelism#Общая очередь и thread pool starvation|thread pool starvation]]. ([34:40](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=2080))

**Асинхронное исполнение** - задачи выполняются не по порядку. Порядок важен внутри одного перевода денег, но не между переводами. ([29:21](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=1761))

Ключевое отличие асинхронности: исполняющий поток не блокируется. ([45:56](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=2756))

**Асинхронное ожидание** - поток откладывает задачу, упёршуюся в блокирующую операцию (сетевой запрос, запись на диск), делает другие задачи и доделывает отложенную, когда операция завершится. ([35:35](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=2135))

Однопоточная асинхронность похожа на синхронные потоки на одном ядре с [[mtap-L01-потоки-и-процессы#Планировщик и кванты|прерываниями]]. Но при прерываниях шаг 4 перевода не начнётся, пока не закончится заблокированный шаг 2, а асинхронность вместо ожидания делает другую работу. ([33:24](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=2004))

### Event loop

По лектору, однопоточная асинхронность устроена как **event loop**: все задачи стоят в одной очереди, поток берёт их по одной, а задачу с блокирующей операцией кладёт в конец и, снова дойдя до неё, проверяет готовность. Так работают Python и Node.js. ([38:26](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=2306))

В asyncio ожидающая задача в очереди не стоит: event loop возобновляет её, когда future завершён (вне конспекта, [docs](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task)).

**Future**, или **promise** - обёртка над работой, которую делает кто-то другой: результата пока нет, но он обещан в будущем. ([40:23](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=2423))

## От ContinueWith к async/await

До `async`/`await` асинхронный код писали на колбэках APM, потом на [[mtap-L06-plinq-и-tpl#ContinueWith|ContinueWith]]: поток не блокируется, но каждая операция добавляет уровень вложенности. ([49:01](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=2941))

```csharp
// ContinueWith: лапша из вложенных продолжений
return File.WriteAllBytesAsync(file, data)
    .ContinueWith(_ =>
    {
        File.ReadAllBytesAsync(file).ContinueWith(readTask =>
        {
            var content = readTask.Result;
            // ...
            File.WriteAllBytesAsync(file, data)
                .ContinueWith(_ => Console.WriteLine("After"), TaskContinuationOptions.AttachedToParent);
        }, TaskContinuationOptions.AttachedToParent);   // без опции родитель не дождётся
    });

// async/await: то же самое
static async Task WriteReadWriteAsync(string file, byte[] data)
{
    await File.WriteAllBytesAsync(file, data);         // поток не блокируется
    var content = await File.ReadAllBytesAsync(file);  // await разворачивает Task<byte[]> в byte[]
    for (var i = 0; i < content.Length; i++)
        content[i] ^= 42;
    await File.WriteAllBytesAsync(file, data);
}
```

`Wait` и `GetAwaiter().GetResult()` на асинхронной задаче блокируют поток и ломают весь выигрыш, нормальный способ ждать - только `await`. ([52:10](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=3130))

## async/await

**await** - асинхронное ожидание задачи: дойдя до блокирующей операции, поток откладывает метод и уходит на другую работу. ([01:04:11](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=3851))

**async** - пометка в сигнатуре, по которой компилятор особым образом собирает метод, чтобы в нём работал `await`. ([01:00:47](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=3647))

Метод с `async` возвращает `Task` или `Task<T>`. Вызывающий тоже ждёт через `await`, поэтому `async` поднимается до `async Task Main`, а синхронному методу остаётся блокирующий `GetAwaiter().GetResult()`. ([01:08:23](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=4103))

Задачу без `await` никто не ждёт: метод идёт дальше, и чтение файла может начаться до конца записи. ([01:02:54](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=3774))

Код до первого `await` исполняется синхронно, а синхронные и асинхронные вызовы в одном методе можно смешивать. ([01:11:32](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=4292))

### Смена потока после await

Код после `await` - продолжение, как у `ContinueWith`, и исполнит его, скорее всего, другой поток пула: исходный ушёл на другие задачи. Код, завязанный на данные потока, после `await` ломается. ([01:04:56](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=3896))

Поэтому `await` внутри `lock` нельзя: критическую секцию исполняет один конкретный поток, а продолжение может достаться другому. ([01:15:19](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=4519)) Компилятор такой код не пропустит, ошибка CS1996 (вне конспекта, [docs](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-messages/async-await-errors)).

### Исключения

`Wait` и `Result` оборачивают исключение в [[mtap-L05-пул-потоков-и-data-parallelism#AggregateException|AggregateException]], а `GetAwaiter().GetResult()` и `await` бросают исходное. Поэтому из двух блокирующих способов в 99% случаев выбирают `GetAwaiter().GetResult()`. ([01:13:12](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=4392))

```csharp
task.Wait();                   // AggregateException
task.GetAwaiter().GetResult(); // исходное исключение, но поток заблокирован
await task;                    // исходное исключение без блокировки
```

### Async снаружи не виден

Синхронный и асинхронный методы снаружи неразличимы: оба возвращают `Task`, а `async` в интерфейсе нет. Суффикс `Async` - соглашение, а не гарантия: внутри может быть блокирующий `GetAwaiter().GetResult()`. ([01:16:52](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=4612))

Брошенную без `await` задачу ловит предупреждение компилятора (CS4014, вне конспекта), и в проектах его часто делают ошибкой. ([01:18:50](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=4730))

## Много асинхронных задач

`Select` с асинхронной лямбдой только запускает задачи, не дожидаясь их, а ждут всех через `await Task.WhenAll`. ([01:20:20](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=4820))

```csharp
var writeTasks = Enumerable.Range(0, 1000)
    .Select(async i => await File.WriteAllTextAsync(i.ToString(), i.ToString()))
    .ToArray();                                   // все записи уже запущены
await Task.WhenAll(writeTasks);                   // не Wait и не WaitAll: те блокируют

var readTasks = Enumerable.Range(0, 10)
    .Select(async i => int.Parse(await File.ReadAllTextAsync(i.ToString())));
int[] results = await Task.WhenAll(readTasks);    // массив результатов
```

Задачу можно не ждать намеренно. В рабочем коде лектора задачи не копятся в список для `WhenAll`: конец обработки ловят, заняв все места семафора, ограничивающего параллелизм. ([01:23:01](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=4981))

```csharp
using var semaphore = new SemaphoreSlim(10, 10);   // не больше 10 задач разом
foreach (var item in items)
{
    await semaphore.WaitAsync();
    _ = Task.Run(async () =>                       // без await намеренно
    {
        try { await ProcessAsync(item); }
        finally { semaphore.Release(); }
    });
}
for (var i = 0; i < 10; i++)
    await semaphore.WaitAsync();   // заняли все 10 мест - значит, ни одна задача не работает
```

По лектору, `WaitAsync` можно вызвать из одного потока 10 раз, а обычный `Wait` здесь дал бы [[mtap-L03-дедлоки-и-cas#Дедлок|дедлок]] нереентерабельного семафора. ([01:26:47](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=5207)) По документации `SemaphoreSlim` не привязан к потоку: `Wait` тоже дождался бы мест, только заблокировав поток (вне конспекта, [docs](https://learn.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim)).

## Когда асинхронность выгодна

Асинхронность не ускоряет код, а лишь не даёт ему блокироваться. Синхронная часть задачи должна быть небольшой, иначе поток работает без переключений и выигрыш пропадает. ([01:30:45](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=5445))

Заблокироваться может любой вызванный метод, поэтому его стоит вызывать асинхронно, если такая версия есть: например, `Task.Delay` вместо `Thread.Sleep`. ([01:29:41](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=5381))

Примеры синхронной части: ([01:31:46](https://kontur.ktalk.ru/recordings/XbUN6UjsbEQprlYUiepg?t=5506))
- UI-поток надо как можно раньше отпустить к действиям пользователя, отдав работу другому воркеру
- одновременный запуск нескольких асинхронных методов тоже занимает время

> [!note]- Пропущено
> повтор L06, демо статусов (продолжение ждёт в `WaitingForActivation`), `Task.FromCanceled` и `FromException` для тестов, аналогия с ответом в Telegram, служебные потоки многопоточной асинхронности, код APM с `BeginWrite` и `AutoResetEvent`, демо `AggregateException` с несколькими дочерними задачами (`Flatten`, `Handle`) из архива, работа над лёгкими потоками в C#, переподключение лектора.
