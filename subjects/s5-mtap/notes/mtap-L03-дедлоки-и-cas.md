---
date: 2026-10-21
ingest: fast
links:
  - https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR
---
# L03. Дедлоки, проблемы блокировок, CAS

Код восстановлен по демо лектора и мог выглядеть на экране иначе.

## Дедлок

**Дедлок** - вечная взаимная блокировка потоков, при которой программа зависает. Разбудить такие потоки снаружи нельзя, поможет только перезапуск программы. ([00:24](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=24))

Классический пример - вложенные блокировки в разном порядке. Это та же ситуация, что в задаче об обедающих философах. ([04:45](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=285))

```csharp
// поток 1: Петя -> Вася
lock (petya)
{
    Thread.Sleep(100);   // даём второму потоку взять свой лок
    lock (vasya) { ... }
}

// поток 2: Вася -> Петя
lock (vasya)
{
    Thread.Sleep(100);
    lock (petya) { ... }
}
// поток 1 держит Петю и ждёт Васю, поток 2 держит Васю и ждёт Петю
```

Исправление - везде брать блокировки в одном порядке:

```csharp
lock (petya)         // всегда сначала Петя, потом Вася
lock (vasya) { ... }
```

"Молитва против дедлоков" работает в любых языках: ([32:10](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=1930))
1. Вложенные блокировки везде брать в одном порядке.
	- порядок задаёт неизменный ключ: ID или имя, но не баланс
	- если блокируешься по строкам, заведи для каждой строки свой объект-лок и бери их в порядке сортировки строк
2. Отпускать блокировки в обратном порядке, как стек.
	- `lock` делает это сам; следить надо при ручных `Mutex` и `Monitor`
	- пример лектора: поток 1 взял A и B, отпустил A и снова ждёт A, держа B; поток 2 тем временем взял A и ждёт B
	- по сути поток 1 захватил A после B, то есть нарушил и правило 1 (вне конспекта)

Можно взять один глобальный лок на все переводы. Дедлоков не будет, но в каждый момент пройдёт только один перевод, а банку с миллионами операций это не подходит. ([11:43](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=703))

Любой выигрыш в скорости оплачивается усложнением кода, а с ним растут и вероятность ошибки, и её цена.

**Самоблокировка** (авто-дедлок) - поток блокирует сам себя, без всякой многопоточности. ([25:52](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=1552))

```csharp
var semaphore = new SemaphoreSlim(1);
semaphore.Wait();
semaphore.Wait();    // семафор не реентерабелен: поток ждёт сам себя вечно

lock (petya)
lock (petya) { }     // lock реентерабелен: поток пропустят
```

Диагностика дедлока: ([16:11](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=971))
- снять дамп процесса: WinDbg или `dotnet-dump collect`
- в дампе видно, какой поток какой лок держит и какого ждёт
- в крупных компаниях настраивают автоматические дампы

## Другие проблемы

**Lock convoy** (проблема медленной бабушки у банкомата) - тяжёлая операция под блокировкой, за которой копится очередь потоков. ([37:22](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=2242))

```csharp
lock (locker) { shared = Fibonacci(40); }   // плохо: считаем под локом

var result = Fibonacci(40);                 // хорошо: считаем до входа
lock (locker) { shared = result; }
```

Под локом оставляют только обновление общих данных, а вычисления, проверки и подготовку выносят наружу.

**Busy waiting** - поток держит лок и в цикле ждёт события. Он ест CPU и не пускает к ресурсу остальных, поэтому лок надо отпустить и зайти снова позже. ([43:29](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=2609))

```csharp
lock (vasya)
{
    while (vasya.Rubles < value) { }   // ждём денег, держа лок
}
```

**Двойная проверка** (у лектора "дабл чек") - условие проверяют до захвата следующего лока и ещё раз внутри, чтобы не ждать лок ради отказа. ([19:06](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=1146))

```csharp
lock (from)
{
    if (from.Rubles < value) return;   // Вася уже заблокирован, денег ему никто не добавит
    lock (to)
    {
        if (from.Rubles < value) return;
        from.Rubles -= value;
        to.Rubles += value;
    }
}
```

**Локальная копия** - разделяемую переменную читают один раз в локальную и дальше работают с копией. ([45:53](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=2753))

```csharp
// поток 2 время от времени делает eventsCount = 0
var events = eventsCount;          // одно чтение
if (events > 0)
    Console.WriteLine(events);     // eventsCount мог обнулиться, events - нет
```

Приём работает без примитивов, но только пока разделяемая переменная одна. Если рядом появляется, например, общая очередь событий, снова нужна синхронизация.

## Неблокирующая синхронизация

Атомарность, lock-free, wait-free и CAS - в "Уже знаю". Лектор подчёркивает: заблокированный поток спит и ничего полезного не делает, а на HTTP-сервере это съедает пропускную способность. ([54:56](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=3296))

`Interlocked` - класс атомарных операций в C#: ([01:08:26](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=4106))
- `CompareExchange` - CAS
- `Add`, `Increment`, `Decrement`
- `Exchange`
- каждая операция атомарна сама по себе, а цепочка из нескольких - уже нет

```csharp
to.Rubles += value;                      // три операции, гонка
Interlocked.Add(ref to.Rubles, value);   // одна атомарная
```

`Interlocked.CompareExchange(ref location, value, comparand)` возвращает значение `location` до сравнения. Если результат равен `comparand`, запись прошла. ([01:02:12](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=3732))

Списание без локов: ([01:10:27](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=4227))

```csharp
while (true)
{
    var localFrom = from.Rubles;            // копия
    if (localFrom < value) return;          // проверка по копии
    if (Interlocked.CompareExchange(ref from.Rubles, localFrom - value, localFrom) == localFrom)
        break;                              // никто не вмешался
}
Interlocked.Add(ref to.Rubles, value);
```

- В CAS передают разделяемое поле, а не локальную копию. ([01:17:53](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=4673))
- На практике цикл прячут в метод-хелпер.
- Так же устроен `ConcurrentDictionary`.

По лектору, `Interlocked` примерно в 10 раз быстрее блокировок. Но он медленнее обычного `+=`, поэтому его используют только там, где нужна атомарность. ([01:25:00](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=5100))

На следующей лекции: нужен ли в цикле таймаут или пауза, замеры скорости и языки, где такой код не работает.

> [!note]- Пропущено
> аналогия с банкоматом, вопрос про `lock` на кортеж (см. [[mtap-L02-блокирующая-синхронизация#lock и Monitor|L02]]), false sharing (лектор не объяснил), `ThreadStatic`/`ThreadLocal` (перенесены).
