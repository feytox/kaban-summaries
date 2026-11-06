---
date: 2026-10-14
ingest: fast
links:
  - https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3
---
# L02. Блокирующая синхронизация

Код восстановлен по демо лектора и мог выглядеть на экране иначе.

## Гонка

**Состояние гонки** (race condition) - результат зависит от порядка, в котором потоки попадают в критическую секцию. ([04:13](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=253))

Память между потоками никто не ограничивает, поэтому доступ к данным синхронизирует сам разработчик.

```csharp
static void SendMoney(Account from, Account to, int value)
{
    if (from.Rubles < value) return;   // проверка
    Thread.Sleep(...);                 // лектор "портит" код, чтобы гонка случалась чаще
    from.Rubles -= value;              // действие отдельно от проверки
    to.Rubles += value;
}
// Вася (100 руб.) -> Петя и Вася -> Саша по 100 в двух потоках: оба прошли проверку, у Васи -100
```

([05:13](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=313))

- Гонка возникает, когда проверка (чтение) и действие (запись) идут отдельными шагами.
- `+=` - тоже три операции: чтение, изменение, запись. Поэтому лок нужен и на получателя. ([25:21](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1521))
	- 100 потоков по `+1` к общей переменной почти никогда не дают 100
	- атомарная замена - `Interlocked.Add` ([[mtap-L03-дедлоки-и-cas#Неблокирующая синхронизация|L03]])

## lock и Monitor

**Критическая секция** (эксклюзивная блокировка) - участок кода, который в каждый момент выполняет только один поток. ([11:50](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=710))

```csharp
lock (from)
lock (to)          // порядок вложенных локов - в L03
{
    if (from.Rubles < value) return;
    from.Rubles -= value;
    to.Rubles += value;
}
```

- `lock` - основной примитив, он нужен в 99% случаев. ([10:09](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=609))
- В `lock` можно передать любой ссылочный тип; пустой `lock` не скомпилируется.
- Что блокирует `lock`, код или объект? Формально критическая секция - это код, а объект служит её именем. Лектор считает верными оба объяснения. ([19:36](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1176))
- Лок привязан к объекту, а не к переменной: переприсвоили переменную - потоки встали в разные очереди.

`lock` - синтаксический сахар над `Monitor`: ([30:41](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1841))

```csharp
bool lockTaken = false;
try
{
    Monitor.Enter(locker, ref lockTaken);   // занято -> поток встаёт в очередь объекта и ждёт
    SendMoney(from, to, value);
}
finally
{
    if (lockTaken) Monitor.Exit(locker);    // отпускаем даже при исключении
}
```

- `finally`: без него исключение внутри оставит лок захваченным, и всё зависнет.
- `lockTaken` нужен из-за `Thread.Abort`: тот выбрасывал `ThreadAbortException` в любой точке потока, даже до входа в монитор. ([32:40](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1960))
- `Monitor.TryEnter` в очередь не встаёт: возвращает `bool`, и поток идёт дальше. Лектор не придумал, где он нужен.
- `Monitor.Enter` в цикле по нескольким объектам захватывает сразу несколько локов; `lock` - частный случай для одного объекта. ([58:12](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3492))

Новый тип `Lock` из C# работает так же, с мелкими отличиями под капотом. Лектор не видит причин его использовать и пишет обычный `lock`; нейросеть тоже выдаст обычный. ([16:26](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=986))

```csharp
private static readonly Lock locker = new();
using (locker.EnterScope()) { ... }
```

`ConcurrentDictionary` берёт лок не на ключ и не на бакет, а на группу бакетов (**striped locking**); чтение идёт без локов. Подробно - в [[mtap-L04-коллекции-и-producer-consumer#ConcurrentDictionary|L04]]. ([21:01](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1261))

## Правила

Объект блокировки: ([46:23](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2783))
- **приватное поле**, общее для всех потоков; лок на локальном объекте бессмыслен: метод не гоняется сам с собой
- **не строка**: строки неизменяемы, любое изменение (`Replace`) с переприсваиванием даёт новый объект ([47:13](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2833))
- **не строковый литерал**: литералы интернируются, и два несвязанных лока на `"123"` в разных классах станут одним ([50:36](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3036))
- **не публичное поле**: его могут переприсвоить, и потоки будут ждать у разных "дверей" ([54:07](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3247))

Лок работает, только если его берут все потоки, которые трогают эти данные. Поток без `lock` пройдёт мимо, как через дыру в стене рядом с закрытой дверью. ([17:45](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1065))

Синхронизация через `Sleep` не работает: испортить код задержкой можно, а починить нельзя. Локально, на билд-агенте и на проде тайминги разные. ([59:20](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3560))

`IsBackground = false`: `Main` при выходе ждёт такой поток, и незавершённый поток подвесит программу. Фоновые потоки рантайм убивает сам. ([05:58](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=358))

## Mutex

**Mutex** - эксклюзивная блокировка на уровне ОС с именем-строкой. Её видят все процессы, в том числе программы на других языках. ([37:21](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2241))

```csharp
using var mutex = new Mutex(initiallyOwned: false, name: "my-app");
if (!mutex.WaitOne(TimeSpan.FromSeconds(1)))   // за секунду не вошли -> false, уходим
    return;
try { ... }                                    // критическая секция
finally { mutex.ReleaseMutex(); }              // try/finally пишем сами
```

- Таймаут в `WaitOne` ограничивает ожидание входа, а не время работы внутри. ([40:04](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2404))
- `initiallyOwned: true` - создавший поток сразу владеет мьютексом, и остальные ждут его `ReleaseMutex`.
- Мьютексов в ОС ограниченное число, как и потоков, поэтому неотпущенный мьютекс опаснее неотпущенного `lock`.
- Отпустить мьютекс может только поток, который его захватил. Лектор так предположил, не будучи уверен; по документации вызов из чужого потока бросает `ApplicationException` (вне конспекта, [docs](https://learn.microsoft.com/en-us/dotnet/api/system.threading.mutex.releasemutex)). ([45:18](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2718))

Пример лектора: приложение на голом железе пишет на диск. При обновлении новая версия стартует рядом со старой, и мьютекс не даёт двум экземплярам одновременно работать с данными. ([41:51](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2511))

## Semaphore и ReaderWriterLock

**Семафор** - многоместная критическая секция: внутрь одновременно пускают до N потоков. Нужен, чтобы ограничить параллелизм, а не для эксклюзивного доступа. ([01:04:35](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3875))

```csharp
var semaphore = new SemaphoreSlim(initialCount: 2, maxCount: 10);
semaphore.Wait();       // свободных мест -1; при 0 поток ждёт
try { ... }
finally { semaphore.Release(); }   // +1, но не выше maxCount
```

- `initialCount` - сколько мест свободно сейчас, `maxCount` - до скольких мест можно дорастить `Release`. ([01:08:03](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4083))
- `(2, 10)` пустит только двоих; все 10 мест сразу даст `(10, 10)`.

**ReaderWriterLock** - много читателей одновременно или один писатель. Писатель блокирует всех, читатели не мешают друг другу. ([01:01:49](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3709))

```csharp
var rw = new ReaderWriterLockSlim();
rw.EnterReadLock();  try { /* читаем */ } finally { rw.ExitReadLock(); }
rw.EnterWriteLock(); try { /* пишем */ } finally { rw.ExitWriteLock(); }
```

- Read lock защищает от писателя посреди долгого чтения, например `foreach` по списку: иначе половина данных окажется старой, половина новой.
- Писатель ждёт, пока выйдут текущие читатели; новых читателей не пускают. ([01:14:08](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4448))
- "Читатель" - только соглашение: запись под read lock не даст ошибки ни при компиляции, ни в рантайме. Нейросеть легко нарушит его, не видя, что код выполняется под read lock. ([01:17:59](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4679))

## Сравнение примитивов

| Примитив               | Сколько потоков внутри         | Между процессами | Реентерабельный |
| ---------------------- | ------------------------------ | ---------------- | --------------- |
| `lock` / `Monitor`     | 1                              | нет              | да              |
| `Mutex`                | 1                              | да               | да              |
| `Semaphore`            | N                              | да               | нет             |
| `SemaphoreSlim`        | N                              | нет              | нет             |
| `ReaderWriterLockSlim` | много читателей или 1 писатель | нет              | -               |

([01:15:39](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4539))

- Slim-версии работают внутри процесса и быстрее обычных.
- **Реентерабельность** - поток, уже держащий блокировку, может войти в неё ещё раз. Пример самоблокировки на семафоре - в [[mtap-L03-дедлоки-и-cas#Дедлок|L03]].
- `ReaderWriterLockSlim`: реентерабельность и межпроцессность лектор не обсуждал.

> [!note]- Пропущено
> аналогия с больницей и дверью, именование переменных через `@lock`, пример масштабирования реплик через семафор, вопрос про порядок локов (разобран в L03), байка про Контур.Толк.
