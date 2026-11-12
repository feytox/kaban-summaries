---
date: 2026-11-11
ingest: fast
links:
  - https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk
  - https://learn.microsoft.com/en-us/dotnet/core/extensions/channels
---
# L05. Пул потоков, data parallelism, Parallel

Код восстановлен по демо лектора, пример с `Parallel.For` частично мой.

## Пул потоков

Создавать и удалять потоки дорого, поэтому их создают заранее и переиспользуют, как и любой пул: буферов, строк. Пул потоков - это [[mtap-L04-коллекции-и-producer-consumer#Producer-consumer|producer-consumer]], где producer - программа, а consumer'ы - воркеры. ([09:06](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=546))

- воркер либо спит, либо работает: без задач он не крутит цикл и не тратит CPU
- свободный воркер берёт новую задачу сразу, без задержек
- по лектору, по умолчанию воркеров столько, сколько ядер, а среда меняет их число сама ([43:55](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=2635))

```csharp
ThreadPool.QueueUserWorkItem(_ => Console.WriteLine(100));
```

Схема та же в любом языке и между сервисами: там очередь - RabbitMQ или Kafka. ([12:46](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=766))

## Общая очередь и thread pool starvation

Если все воркеры и producer'ы работают с одной общей очередью, она становится бутылочным горлышком: в неё пишут и программа, и сами воркеры. ([16:59](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=1019))

**Thread pool starvation** - все воркеры пула заняты, а задачи копятся. По лектору, бывает в любом пуле, не только с одной очередью. ([18:19](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=1099))

| Признаки                                 | Причины                                    |
| ---------------------------------------- | ------------------------------------------ |
| задачи долго исполняются, растёт latency | потоки блокируются при обработке задач     |
| воркеров много, и добавление не помогает | большие накладные расходы на мелкие задачи |
| CPU потребляется мало                    |                                            |

- блокировка: воркеры ждут сеть или диск, например сервис-прослойка ждёт ответа медленного соседа ([21:35](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=1295))
- мелкие задачи: полезная работа - единицы процентов, остальное - взять задачу и переключить контекст ([22:41](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=1361))

## Локальные очереди и work stealing

Локальная очередь у каждого воркера снимает нагрузку с общей, но задачи распределяются неравномерно: один воркер завален, другие простаивают. Поэтому соседи воруют задачи из чужих локальных очередей, в C# это `WorkStealingQueue`. ([24:42](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=1482))

Схема C# `ThreadPool`: ([28:49](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=1729))
1. Внешний поток кладёт задачи в общую очередь.
2. Воркеры кладут порождённые задачи в свою локальную очередь.
3. Воркер ищет задачу:
	1. в своей локальной очереди
	2. в общей очереди
	3. в локальных очередях других воркеров
	4. не нашёл - засыпает до появления задачи в любой очереди

Узнать о новой задаче можно двумя способами: ([34:54](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=2094))
- polling: периодически спрашивать очередь; тратит CPU и нагружает очередь, даже со `SpinWait`
- событие: спать, пока не разбудят, как `Monitor.Wait`/`Pulse`

## Data parallelism и task parallelism

Работу ускоряют двумя способами: клонируют весь конвейер или раздают воркерам его этапы. ([45:36](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=2736))

| Подход           | Как делят работу                    | Задачи                                    | Пример                 |
| ---------------- | ----------------------------------- | ----------------------------------------- | ---------------------- |
| data parallelism | каждый воркер делает задачу целиком | однотипные, дорогие по CPU, слабо связаны | парсинг логов, матрицы |
| task parallelism | у каждого воркера свой этап         | мелкие, зависят друг от друга или от IO   | веб-сервер             |

- дорогие по CPU задачи нужны data parallelism из-за starvation на мелких задачах ([47:32](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=2852))
- в task parallelism этап сборки ждёт предыдущие: задачи образуют граф зависимостей
- эта лекция про data parallelism, остальные - про task parallelism и асинхронность

## Parallel.Invoke, For, ForEach

`Parallel` раздаёт работу потокам пула и загружает их все, программа здесь - producer. ([54:00](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=3240))

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 2,      // не убить себя и сервис, куда ходим
    CancellationToken = cts.Token,   // мягкая остановка снаружи: cts.Cancel()
};
var processed = 0;
var result = Parallel.For(0, 100, options, (i, state) =>
{
    if (i == 10) state.Break();             // или state.Stop()
    Interlocked.Increment(ref processed);   // processed++ здесь - гонка
});
// result.IsCompleted == false, result.LowestBreakIteration == 10
```

- **CancellationToken** - сигнал остановки вместо устаревшего `Thread.Abort`, который бросал `ThreadAbortException` посреди кода; реагировать на сигнал не обязательно, но лучше ([56:53](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=3413))
- синхронизация по-прежнему нужна; запись в массив по разным индексам безопасна ([01:08:48](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=4128))

`Break` и `Stop` повторяют семантику обычного `break`, насколько позволяет параллельность: уже начатые итерации остановить нельзя. ([01:11:25](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=4285))

| Вызов     | Итерации с большим номером | Итерации с меньшим номером        |
| --------- | -------------------------- | --------------------------------- |
| `Break()` | не начнутся                | выполнятся, даже если не начались |
| `Stop()`  | не начнутся                | не начнутся                       |

- `LowestBreakIteration` - наименьший индекс, где вызвали `Break`; все итерации до него точно выполнены ([01:15:28](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=4528))
- `state.ShouldExitCurrentIteration` сообщает, что соседняя итерация попросила остановиться ([01:20:14](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=4814))
- исключение в итерации работает как `Stop`: остальные потоки тоже прекращают работу ([01:19:01](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=4741))

### AggregateException

Исключения из параллельных действий не долетают как есть: `Parallel` собирает их в обёртку `AggregateException`, а настоящие лежат в `InnerExceptions`. ([01:04:38](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=3878))

```csharp
try
{
    Parallel.Invoke(
        () => throw new ArgumentException(),
        () => throw new InvalidOperationException(),
        () => Console.WriteLine(3));
}
catch (ArgumentException) { }   // не поймает
catch (AggregateException e)
{
    foreach (var inner in e.InnerExceptions)
        Console.WriteLine(inner.GetType());
}
```

### Partitioners

**Partitioner** - способ разбить данные между воркерами: range, chunk, striped, hash. ([01:00:33](https://kontur.ktalk.ru/recordings/qWENqPZFCCuZJJQYSMUk?t=3633))

По лектору, разбиением управляет `TaskScheduler` в `ParallelOptions`. По документации `TaskScheduler` решает, на каких потоках выполнять задачи, а разбиение задают объектом `Partitioner`, который передают в `Parallel.ForEach` ([документация](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/custom-partitioners-for-plinq-and-tpl)) (вне конспекта).

На следующей лекции: PLINQ.

> [!note]- Пропущено
> повтор `Monitor.Wait`/`Pulse` из L04, вопросы из чата про дубликаты задач и порядок поиска, аналогия с походом на рынок, каналы C# и `BlockingCollection` (разобрать самостоятельно), история с выкачкой ключей из базы, демо `ResetEvent` и `PulseBeforeWait` из архива примеров, вывод демо с потоками `Parallel.For`.
