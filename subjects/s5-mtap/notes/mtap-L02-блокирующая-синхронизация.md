---
date: 2026-10-14
ingest: fast
links:
  - https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3
---
# L02. Блокирующая синхронизация

Код восстановлен по демо лектора и мог выглядеть на экране иначе.

## Гонка

Язык не ограничивает доступ потоков к общей памяти, синхронизировать их должен разработчик. ([04:13](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=253))

**Состояние гонки** (race condition) - ситуация, когда результат зависит от порядка потоков и от того, кто первым войдёт в критическую секцию.

Демо: у Васи 100 руб., два потока одновременно переводят по 100 Пете и Саше. ([05:13](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=313))

```csharp
static void SendMoney(Account from, Account to, int value)
{
    if (from.Rubles < value) return;   // оба потока проходят проверку
    Thread.Sleep(10);                  // лектор так подгоняет гонку
    from.Rubles -= value;
    to.Rubles += value;
}
// итог: Вася -100, Петя 100, Саша 100
```

Проверка и действие идут отдельно, поэтому между ними вклинивается другой поток. `+=` тоже гонка: это три операции, чтение, сложение и запись.

**Критическая секция** - участок кода, который в один момент выполняет только один поток. Лектор использует как синонимы "эксклюзивная блокировка" и "лок". ([11:50](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=710))

## lock и Monitor

`lock` закрывает 99% случаев. Он защищает данные, только если все потоки, которые их трогают, заходят через него: дверь не поможет, если рядом дыра в стене. ([10:09](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=609))

```csharp
lock (from)        // лектор блокируется на самих объектах счетов
lock (to)
{
    if (from.Rubles < value) return;
    from.Rubles -= value;
    to.Rubles += value;
}
```

Объект блокировки: ([46:23](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2783))
- только ссылочный тип: value type (например, кортеж) каждый раз новый, и блокировка ничего не защищает
- общий для потоков: поле класса, а не локальная переменная
- приватное поле: публичное могут перезаписать, и потоки разойдутся по разным объектам
- не строка
	- изменение строки создаёт новый объект
	- литералы интернируются, и несвязанные блокировки на одинаковых литералах становятся одной
- обычно это `private static readonly object` или поле типа `Lock`

```csharp
private string key = "a:b";
lock (key) { key = key.Replace(':', ';'); }   // следующий поток заблокируется уже на другой строке

private static string a = "123";              // в одном классе
private static string b = "123";              // в другом классе
lock (a) { ... }
lock (b) { ... }                              // это та же блокировка, что и lock (a)
```

В C# 13 появился отдельный тип `Lock`. Лектор не нашёл причин предпочесть его обычному `lock (obj)`. ([16:26](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=986))

```csharp
private readonly Lock locker = new();
using (locker.EnterScope()) { ... }
```

`lock` - синтаксический сахар над `Monitor`: ([30:41](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=1841))

```csharp
bool lockTaken = false;
try
{
    Monitor.Enter(from, ref lockTaken);   // занято - поток встаёт в очередь и ждёт
    SendMoney(from, to, value);
}
finally
{
    if (lockTaken) Monitor.Exit(from);
}
```

- `finally` отпускает блокировку, даже если внутри вылетело исключение.
- Флаг `lockTaken` защищал от `ThreadAbortException` из старого `Thread.Abort`.
- `Monitor.TryEnter` не ждёт, а сразу возвращает `bool`; нужен редко. ([34:42](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2082))
- Через `Monitor` можно захватить несколько объектов, вызывая `Enter` в цикле.

## Mutex, Semaphore, ReaderWriterLock

| Примитив         | Мест | Между процессами       | Реентерабельный |
| ---------------- | ---- | ---------------------- | --------------- |
| `lock`/`Monitor` | 1    | нет                    | да              |
| `Mutex`          | 1    | да, объект ОС по имени | да              |
| `Semaphore`      | N    | да                     | нет             |
| `SemaphoreSlim`  | N    | нет, зато быстрее      | нет             |

([37:21](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2241))

**Реентерабельность** - поток, который уже держит блокировку, может взять её ещё раз. ([01:16:25](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=4585))

### Mutex

```csharp
using var mutex = new Mutex(initiallyOwned: false, name: "my-app-disk");
if (!mutex.WaitOne(TimeSpan.FromSeconds(1))) return;   // за секунду не дождались
try { /* критическая секция */ }
finally { mutex.ReleaseMutex(); }
```

- Мьютекс живёт в ОС, по имени его находят другие процессы, даже на других языках.
- `try/finally` и освобождение пишешь сам, `lock` за тебя этого не сделает.
- `initiallyOwned: true` создаёт мьютекс сразу захваченным.
- Число мьютексов в ОС ограничено.
- Отпустить чужой мьютекс нельзя: `ReleaseMutex` из потока-невладельца бросает `ApplicationException` (вне конспекта, [docs](https://learn.microsoft.com/en-us/dotnet/api/system.threading.mutex.releasemutex)).
- Если поток завершился, не отпустив мьютекс, следующий захвативший получит `AbandonedMutexException` (вне конспекта, там же).

Пример лектора: приложение на голом железе пишет на диск, а при деплое старая и новая версии какое-то время работают рядом. Именованный `Mutex` не даёт двум экземплярам одновременно работать с диском. ([41:51](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=2511))

### Semaphore

```csharp
var semaphore = new SemaphoreSlim(initialCount: 2, maxCount: 10);
semaphore.Wait();                   // свободных мест 2 -> 1; при 0 поток ждёт
try { ... }
finally { semaphore.Release(); }    // +1 место, но не больше maxCount
```

([01:04:35](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3875))

- `initialCount` - сколько мест свободно сейчас.
- `maxCount` - до скольких мест можно дорелизить семафор.
- Применение - ограничить параллелизм, например не больше 10 потоков одновременно.
- Для эксклюзивного доступа семафор не подходит: он пускает несколько потоков.

### ReaderWriterLockSlim

```csharp
var rw = new ReaderWriterLockSlim();

rw.EnterReadLock();
try { Read(); } finally { rw.ExitReadLock(); }

rw.EnterWriteLock();
try { Write(); } finally { rw.ExitWriteLock(); }
```

([01:01:07](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3667))

- Читатели проходят одновременно, писатель работает один.
- Когда приходит писатель, новые читатели ждут, а он ждёт, пока уйдут текущие.
- Читательский лок нужен, чтобы читатель не увидел промежуточное состояние, которое оставил писатель.
- Писать под read-локом язык не запретит: не будет ни ошибки компиляции, ни исключения.

## Правила

- Синхронизация через `Sleep` и задержки не работает: на ноутбуке, CI-агенте и проде тайминги разные. Сломать код задержкой можно, а починить нельзя. ([59:20](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=3560))
- Если у незавершённого потока `IsBackground = false`, программа не выйдет из `Main` и зависнет. При `true` рантайм сам убьёт поток, когда `Main` закончится. ([05:58](https://kontur.ktalk.ru/recordings/mY563QzooY8BgJaaRej3?t=358))

Пропущено: подробности демо с переводами, `@lock` как имя переменной, устройство `ConcurrentDictionary` (будет позже), пример с масштабированием реплик через семафор.
