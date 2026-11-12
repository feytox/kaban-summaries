---
date: 2026-10-28
ingest: full
links:
  - https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW
  - https://habr.com/ru/companies/skbkontur/articles/348508/
---
# L04. Особенности CAS, атомарность, конкурентные коллекции, producer-consumer

Код восстановлен по демо лектора и мог выглядеть на экране иначе.

## CAS-цикл: прогресс и SpinWait

Потоки в [[mtap-L03-дедлоки-и-cas#Неблокирующая синхронизация|CAS-цикле]] не могут вечно обгонять друг друга: неудачный CAS значит, что другой поток записал и вышел. Конкурентов становится меньше, если только новые не приходят быстрее. ([03:04](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=184))

Зато цикл без пауз - это [[mtap-L03-дедлоки-и-cas#Другие проблемы|busy waiting]]. `Thread.Sleep` не подходит: он блокирует поток, а число потоков ограничено. Стандартное решение - `SpinWait`: ([06:21](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=381))

```csharp
var spinWait = new SpinWait();
while (true)
{
    var fromRubles = from.Rubles;
    if (fromRubles < value) return;
    if (Interlocked.CompareExchange(ref from.Rubles, fromRubles - value, fromRubles) == fromRubles)
        break;
    spinWait.SpinOnce();   // exponential backoff
}
```

`SpinOnce` с каждой неудачей ждёт чуть дольше, как TCP наращивает окно. Усыпить поток он может только в крайнем случае, при очень высокой конкуренции.

## Цена Interlocked

По скорости: обычная инструкция быстрее `Interlocked`, а он быстрее `lock`. Взятие блокировки стоит около 25 нс, обращение к кэшу для CAS около 0,5 нс. ([19:22](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=1162))

`Interlocked` не бесплатный, поэтому его не ставят "на всякий случай" в однопоточный код. Вычитания в нём нет, прибавляют отрицательное: `Interlocked.Add(ref x, -value)`.

## ABA и use-after-free

**ABA-проблема** - CAS видит прежнее значение и решает, что ничего не менялось, хотя структуру успели изменить и вернуть то же значение. ([11:53](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=713))

Пример со стеком `A B C`: поток 1 в `TryPop` запомнил голову `A` и потерял квант. Поток 2 сделал `Pop A`, `Pop B`, `Push A`. Поток 1 видит `A` в голове и продолжает, хотя стек уже `A C`.

- лечится версионированием: рядом со значением хранят версию и увеличивают её при каждом изменении
- по лектору, в C# ABA нет: CAS сравнивает ссылки на конкретные объекты, а не указатели на память, как в C++

**Use-after-free** - обращение к уже освобождённой памяти. В C++ возможно в таком же CAS-цикле на сырых указателях; спасают указатели со счётчиком ссылок. ([17:16](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=1036))

## Атомарность чтения и записи

Процессор читает и пишет память машинными словами: 32 или 64 бита. Значение длиннее слова пишется в несколько операций, значит неатомарно. ([24:11](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=1451))

| Тип              | 32 бит   | 64 бит   |
| ---------------- | -------- | -------- |
| `int`, ссылка    | атомарно | атомарно |
| `long`, `double` | нет      | атомарно |
| структура        | нет      | нет      |

По лектору, ссылка в C# занимает 4 байта. На 64-битной системе она занимает 8, но спецификация всё равно гарантирует атомарность записи ссылок ([спецификация, 9.6](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/variables#96-atomicity-of-variable-references)) (вне конспекта).

`long` на 32 битах читают и пишут через `Interlocked.Read` и `Interlocked.Exchange`. ([28:34](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=1714))

Структура - несколько слов, поэтому целиком атомарно её не записать. Вдобавок C# выравнивает поля: у лектора структура из `byte` и `long` легла как `| 0 0 0 byte | long |`. ([31:23](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=1883))

Выход - обернуть структуру в класс и подменять ссылку на объект целиком. Так устроены и конкурентные коллекции. ([32:30](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=1950))

Про массив:
```csharp
arr = new byte[10];                    // атомарно: одна ссылка
for (var i = 0; i < arr.Length; i++)   // не атомарно, гонка
    arr[i] = 1;
```

## Конкурентные коллекции

`ConcurrentQueue` и `ConcurrentStack` повторяют обычные коллекции, а голову подменяют через CAS. ([34:23](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=2063))

По лектору, `ConcurrentBag` заменил `ConcurrentDictionary<T, byte>` в роли множества. На деле `ConcurrentBag` не множество: он допускает дубликаты ([документация](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentbag-1)) (вне конспекта).

### ConcurrentDictionary

Внутри массив bucket'ов, а в каждом bucket'е связный список узлов. На каждый ключ создаётся лишний объект, и растёт нагрузка на GC. ([35:13](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=2113))

**Striped locking** - небольшой массив локов, где каждый отвечает за свой диапазон bucket'ов. ([36:36](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=2196))
- писатели блокируют друг друга, только если попали в один диапазон
- чтение lock-free всегда, даже во время записи; это не [[mtap-L02-блокирующая-синхронизация#ReaderWriterLock|ReaderWriterLock]]

Сложность для N элементов и K локов, лектор просил запомнить: ([статья](https://habr.com/ru/companies/skbkontur/articles/348508/))

| Операция                           | Время | Память | Локов |
| ---------------------------------- | ----- | ------ | ----- |
| `ContainsKey`, `TryGetValue`       | O(1)  | O(1)   | 0     |
| `TryAdd`, `TryRemove`, `TryUpdate` | O(1)  | O(1)   | 1     |
| `Clear`                            | O(K)  | O(K)   | K     |
| `Count`, `IsEmpty`                 | O(K)  | O(1)   | K     |
| `Keys`, `Values`                   | O(N)  | O(N)   | K     |
| перечисление                       | O(N)  | O(1)   | 0     |

- `Keys` и `Values` берут все локи и копируют данные в новый `List`, поэтому ключи лучше брать перечислением: `dict.Select(p => p.Key)`
- словарь можно менять прямо в `foreach`, например вычищать устаревшие записи кэша ([45:17](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=2717))

`TryUpdate` работает под локом без цикла, а в `AddOrUpdate` внутри `while (true)`: если ключ успели изменить, он пробует снова. Поэтому фабрика может выполниться несколько раз, и побочные эффекты в ней испортят данные. ([42:59](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=2579))

```csharp
dict.AddOrUpdate(1, key => 2, (key, oldValue) => oldValue + 1);  // фабрики могут вызваться повторно
```

По статье, то же верно для `GetOrAdd`; если повтор недопустим, значение оборачивают в `Lazy<T>`.

## Producer-consumer

**Producer-consumer** - паттерн, где producer ставит задачи, а ограниченное число consumer'ов (worker'ов) их выполняет. Между ними буфер: очередь в программе или брокер между сервисами. ([51:40](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=3100))

Требования: ([55:03](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=3303))
- producer быстрый: продюсеры не блокируют друг друга
- consumer'ы не простаивают
- задачи берут по выбранному правилу
	- в порядке постановки в очередь
	- в порядке готовности: результат отдают сразу, и медленная задача не держит остальных

### ResetEvent

Барьер, который открывают и закрывают вручную, как лампочку над кабинетом врача. ([59:28](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=3568))

| Событие            | `Set()` пропускает | Закрывается             |
| ------------------ | ------------------ | ----------------------- |
| `AutoResetEvent`   | один поток         | само после прохода      |
| `ManualResetEvent` | всех, кто ждёт     | вручную через `Reset()` |

- критическую секцию всё равно делают на `lock`: он безопаснее ([01:05:17](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=3917))
- зато событием управляют снаружи: можно копить потоки час и пустить всех разом, чего не умеет [[mtap-L02-блокирующая-синхронизация#Semaphore|семафор]]
- нужны редко

### Monitor.Wait и Pulse

Если consumer ждёт события под локом, остальные consumer'ы стоят. Лок можно временно отпустить через `Monitor`: ([01:11:09](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=4269))
- `Monitor.Wait(obj)` отпускает лок и ставит поток в очередь ожидающих
- `Monitor.Pulse(obj)` будит одного, `Monitor.PulseAll(obj)` - всех
- проснувшийся поток снова берёт лок, когда его отпустит разбудивший, так что критическая секция сохраняется

```csharp
var queue = new Queue<int>();

void Worker()
{
    lock (queue)
    {
        while (true)
        {
            if (queue.TryDequeue(out var number))
                Console.WriteLine(number);
            else
                Monitor.Wait(queue);   // отпускаем лок, ждём задачу
        }
    }
}

void Produce(int task)
{
    lock (queue)
    {
        queue.Enqueue(task);
        Monitor.Pulse(queue);          // будим одного worker'а
    }
}
```

Вариант с `ConcurrentQueue` и `SpinWait` тоже работает, но там worker засыпает сам и сам решает, когда проснуться. С `Pulse` продюсер будит worker'а, как только появилась задача. ([01:19:36](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=4776))

Здесь обычная `Queue`: лок уже есть, а конкурентная коллекция стоила бы скорости и памяти. ([01:23:18](https://kontur.ktalk.ru/recordings/02Z8x6tNtCZmPk65KhyW?t=4998))

На следующей лекции: пул потоков.

> [!note]- Пропущено
> аналогия с TCP подробно, `Volatile.Write` в исходниках словаря, детали статьи (`concurrencyLevel`, удаление по паре ключ-значение, перезапись `Guid`), демо `ResetEvent`, `ManualResetEventSlim`.
