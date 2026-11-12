---
date: 2026-10-14
ingest: fast
links:
  - https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3
---
# L02. Гонки и блокирующая синхронизация

Код восстановлен по демо лектора и мог выглядеть на экране иначе.

## Гонка и критическая секция

Язык и ОС не ограничивают потоки одного процесса в доступе к общей памяти: пускать к данным только один поток значило бы душить многоядерный процессор. Поэтому доступ синхронизирует сам разработчик. ([03:29](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=209))

**Гонка** (race condition) - ситуация, когда результат программы зависит от порядка, в котором потоки попадают в критическую секцию. ([04:13](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=253))

```csharp
static void SendMoney(Account from, Account to, int value)
{
    if (from.Rubles < value) return;   // чтение и решение
    from.Rubles -= value;              // запись: другой поток мог уже списать деньги
    to.Rubles += value;                // += тоже три операции: чтение, сложение, запись
}
// у Васи 100 руб., два потока переводят по 100 Пете и Саше:
// оба проходят проверку, у Васи -100
```

Гонка возникает, когда проверка отделена от действия: к моменту записи прочитанное значение уже устарело. ([09:32](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=572))

**Критическая секция** (эксклюзивная блокировка) - участок кода, который в каждый момент выполняет только один поток. ([11:50](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=710))

```csharp
lock (from)      // проверку и списание никто не разорвёт
lock (to)        // += на получателе тоже защищаем
{
    if (from.Rubles < value) return;
    from.Rubles -= value;
    to.Rubles += value;
}
```

Порядок вложенных локов разобран в [[mtap-L03-дедлоки-и-cas#Дедлок|L03]]. ([24:40](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1480))

Синхронизация через `Sleep` и задержки не работает. Сломать код задержкой можно, починить нельзя: на ноутбуке, CI-агенте и проде тайминги разные. ([59:20](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3560))

## lock и Monitor

`lock` - основной примитив синхронизации, по лектору это 99% случаев. Он лишь синтаксический сахар над `Monitor`, а захват стоит в `try/finally`, чтобы исключение не оставило лок занятым навсегда. ([30:41](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1841))

```csharp
bool lockTaken = false;
try
{
    Monitor.Enter(vasya, ref lockTaken);   // если занято, поток встаёт в очередь объекта и ждёт
    SendMoney(vasya, petya, 100);
}
finally
{
    if (lockTaken) Monitor.Exit(vasya);
}
```

- Флаг `lockTaken` нужен из-за `ThreadAbortException`: `Thread.Abort` мог выбросить его в любой точке, даже до захвата.
	- в .NET 5+ `Thread.Abort` не поддерживается (вне конспекта, [docs](https://learn.microsoft.com/en-us/dotnet/api/system.threading.thread.abort))
- `Monitor.TryEnter` не встаёт в очередь: сразу возвращает `bool`, и поток идёт дальше.
- Через `Monitor` можно захватить несколько объектов, например в цикле.
- `new Lock()` с `EnterScope()` из нового C# делает то же самое, но лектор пишет обычный `lock`.

Лок защищает, только если через него ходят все потоки: поток без `lock` пройдёт "через дыру в стене рядом с дверью". ([17:45](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1065))

## Объект блокировки

Объект в `lock` - имя критической секции. Потоки исключают друг друга, только если блокируются на одном и том же объекте, поэтому это общее приватное поле ссылочного типа. ([46:23](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2783))

```csharp
private static object locker = new();   // так

lock (new object()) { ... }             // у каждого вызова свой объект, секции нет

lock (text)
{
    text = text.Replace(":", " ");      // строка неизменяема: следующий поток блокируется на новой
}

static string a = "123", b = "123";
lock (a) { ... }                        // литералы интернированы: a и b - один объект,
lock (b) { ... }                        // и несвязанные секции блокируют друг друга

public static object Locker = new();    // кто-то сделает Locker = new(), и потоки разойдутся
```

## Mutex

**Mutex** - объект ОС для эксклюзивной блокировки. Мьютекс с именем видят все процессы, даже написанные на других языках. ([37:21](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2241))

```csharp
using var mutex = new Mutex(initiallyOwned: false, "my-app");   // true: создатель сразу владеет
if (!mutex.WaitOne(TimeSpan.FromSeconds(1)))   // сколько ждать входа, а не сколько держать
    return;
try { WriteToDisk(); }
finally { mutex.ReleaseMutex(); }              // try/finally пишут сами
```

- Мьютексов в ОС ограниченное число, поэтому неосвобождённый мьютекс опаснее забытого лока.
- Лектор предположил, что отпустить мьютекс может только поток-владелец. Документация это подтверждает: иначе `ApplicationException` (вне конспекта, [docs](https://learn.microsoft.com/en-us/dotnet/api/system.threading.mutex.releasemutex)).

Пример лектора: при обновлении на голом железе второй экземпляр приложения не должен писать на тот же диск. ([41:51](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2511))

## Semaphore и ReaderWriterLock

Не всякой задаче нужна эксклюзивная блокировка: семафор пускает внутрь N потоков, а reader-writer lock не блокирует читателей друг другом.

**Семафор** - многоместная критическая секция. Он ограничивает параллелизм, но от одновременной записи не защищает. ([01:04:35](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3875))

```csharp
var semaphore = new SemaphoreSlim(initialCount: 2, maxCount: 10);
semaphore.Wait();                    // свободных мест -1; при 0 поток ждёт
try { Work(); }
finally { semaphore.Release(); }     // +1 место, но не выше maxCount
```

`initialCount` - сколько мест свободно сейчас, `maxCount` - до скольких мест можно дорастить семафор вызовами `Release`. ([01:09:48](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4188))

- `(2, 10)` пустит только двоих, все 10 мест сразу даст `(10, 10)`.

**Reader-writer lock** - блокировка, под которую читатели заходят вместе, а писатель только один и без читателей. ([01:01:49](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3709))

```csharp
var rw = new ReaderWriterLockSlim();
rw.EnterReadLock();  try { Read(); }  finally { rw.ExitReadLock(); }
rw.EnterWriteLock(); try { Write(); } finally { rw.ExitWriteLock(); }
```

Читатели берут лок, чтобы не увидеть промежуточное состояние, пока писатель меняет данные. Пришедший писатель не пускает новых читателей и ждёт, пока выйдут текущие. ([01:14:08](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4448))

Писать под read-lock никто не запретит: ни компилятор, ни рантайм ошибки не дадут. ([01:18:28](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4708))

| Примитив       | Потоков внутри | Между процессами | Реентерабельный |
| -------------- | -------------- | ---------------- | --------------- |
| lock / Monitor | 1              | нет              | да              |
| Mutex          | 1              | да               | да              |
| SemaphoreSlim  | N              | нет              | нет             |
| Semaphore      | N              | да               | нет             |

([01:15:39](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4539))

- Slim-версии работают только внутри процесса, зато быстрее.

**Реентерабельность** - возможность потоку, который уже держит блокировку, взять её ещё раз (пример в [[mtap-L03-дедлоки-и-cas#Дедлок|L03]]). ([01:16:25](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4585))

> [!note]- Пропущено
> кольца защиты ОС, `IsBackground` (перенесено в [[mtap-L01-потоки-и-процессы|L01]]), аналогия с больницей, опыт с `+= 1` в 100 потоках на Python, вопрос про `ConcurrentDictionary` и striped locking (см. [[mtap-L04-коллекции-и-producer-consumer#ConcurrentDictionary|L04]]), имена переменных через `@` (`@lock`), пример семафора для масштабирования реплик.
