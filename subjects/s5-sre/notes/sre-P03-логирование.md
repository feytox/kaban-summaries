---
date: 2026-10-02
ingest: full
links:
  - https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw
---
# P03. Логирование

Примеры кода частично со слайдов, частично мои.

Логи - фундамент наблюдаемости: от их качества зависит, сколько часов уйдёт на диагностику. В разобранных сбоях Cloudflare, Datadog и Knight Capital сигналы были, но не хватило правильных логов и алертов. ([05:01](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=301))

## Уровни

([06:03](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=363))

| Уровень | Когда                                                   |
| ------- | ------------------------------------------------------- |
| Fatal   | приложение не может продолжать работу                   |
| Error   | пользователь не получил услугу из-за внутренней ошибки  |
| Warn    | неожиданный ответ внешней системы, сбой обойдён ретраем |
| Info    | редкие штатные события: старт, перечитали конфиг        |
| Debug   | вызовы внешних систем, обработка запросов               |
| Trace   | всё подряд                                              |

- уровень обязателен в каждой записи: без него не понять, насколько всё плохо
- на проде не включают Debug и Trace: слишком большая нагрузка на сборщики
- уровень задают переменной окружения при деплое: локально `debug`, на проде `info` ([58:40](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=3520))

## Формат и поля

| Формат | Плюсы                              | Минусы                                     |
| ------ | ---------------------------------- | ------------------------------------------ |
| текст  | читается как есть                  | парсинг regex'ом ломается от правки текста |
| logfmt | читаемо, на 30-40% компактнее JSON | плоский, без вложенности                   |
| JSON   | парсится почти всем, вложенность   | чуть больше объём                          |

JSON - для прода, текст и logfmt - для локальной разработки. ([09:06](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=546))

Хороший лог по чек-листу семинара; имена полей из OpenTelemetry Semantic Conventions: ([11:24](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=684))

```json
{
  "timestamp": "2026-10-02T15:10:03.123Z",  // ISO 8601, UTC, до мс
  "level": "error",
  "message": "Order {order_id} failed",     // шаблон: одинаковые события группируются
  "order_id": 42,                           // уникальные значения - отдельными полями
  "service.name": "orders-api",
  "service.version": "1.4.2",
  "deployment.environment.name": "prod",    // окружение; в большой компании дописывает коллектор
  "k8s.pod.name": "orders-api-7c9f-x2k",    // какая реплика
  "trace_id": "4bf92f35...",                // склеить все строки одного запроса
  "request_id": "b3a1c2e4",
  "duration_ms": 120,                       // где осмысленно
  "error.message": "payment timeout"
}
```

- `"Order 42 failed for request b3a1c2e4"` в тексте - антипаттерн: события не сгруппировать без regex ([14:05](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=845))
- по семинаристу, окружение и ДЦ в стартапе пишут сами, в большой компании их добавляет коллектор ([01:04:19](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=3859))

**Кардинальность** - число уникальных значений поля или лейбла. ([15:04](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=904))

|                | Метрики                                 | Логи и трейсы                          |
| -------------- | --------------------------------------- | -------------------------------------- |
| Природа        | состояние через равные интервалы        | отдельные события                      |
| Кардинальность | должна быть низкой                      | высокая (`request_id`) - норма         |
| Если высокая   | взрыв числа временных рядов, OOM у TSDB | колоночные хранилища под это и сделаны |

## Чувствительные данные

([17:16](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=1036))

```json
// плохо
{"card_number": "4111111111111234", "cvv": "123", "password": "S3cr3tPass!"}
// хорошо: PAN по правилу first6last4, CVV нельзя хранить вообще, даже маскированным
{"card_number": "411111******1234", "password": "[REDACTED]"}
```

Не логировать:
- секреты: пароли, токены, ключи, строки подключения к БД (`postgres://user:pass@...` в тексте ошибки)
- CVV, PIN; номер карты - максимум первые 6 и последние 4 цифры
- персональные и медицинские данные, email в URL
- session id как есть: по нему входят в чужой аккаунт, поэтому вместо него пишут хэш
- исходный код: попадает со stack trace, если в коде зашит пароль

| Техника              | Пример или когда                                 |
| -------------------- | ------------------------------------------------ |
| удаление             | убрать поле или `[REDACTED]`                     |
| частичная маскировка | `411111******1234`                               |
| хэширование          | email: значение скрыто, записи можно сопоставить |
| токенизация          | оригинал в Vault, в логе токен; редко            |
| шифрование поля      | шифруется только чувствительное поле             |
| deny-list в логгере  | запрещённое поле не залогировать даже случайно   |

([19:41](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=1181))

Маскировать можно в логгере приложения или в коллекторе. Коллектор - только последний рубеж: до него секрет уже лежит в логе на ноде. ([22:48](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=1368))

```toml
[transforms.mask_pii]         # Vector: маскирование до хранилища
type = "remap"
inputs = ["source"]
source = 'redact!(., filters: ["us_social_security_number"])'
```

## Сбор и доставка

Логи в файлах на серверах и grep работают, пока сервер один; дальше их собирают в центральное хранилище с поиском. ([25:43](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=1543))

- агенты по ресурсам: Fluent Bit ≈ Vector < Filebeat ≈ OTel Collector < Fluentd < Logstash ([28:05](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=1685))
- **OTLP** - протокол OpenTelemetry сразу для логов, метрик и трейсов, не привязан к вендору
- агент - конвейер input -> filter -> output; так устроены и Logstash, и Vector из ДЗ ([30:40](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=1840))

```toml
# Vector как DaemonSet -> Victoria Logs
[sources.k8s]
type = "kubernetes_logs"                 # читает файлы /var/log/pods на своей ноде

[transforms.parse]                       # строка JSON -> поля
type = "remap"
inputs = ["k8s"]
source = '''
parsed = object!(parse_json!(.message))
. = merge(., parsed)
'''

[sinks.vlogs]
type = "elasticsearch"                   # VL принимает Elasticsearch bulk API
inputs = ["parse"]
endpoints = ["http://victoria-logs:9428/insert/elasticsearch/"]
api_version = "v8"
healthcheck.enabled = false

[sinks.vlogs.query]                      # какие поля считать служебными, см. ниже
_msg_field = "message"
_time_field = "timestamp"
_stream_fields = "service.name"
```

### Доставка в Kubernetes (вне конспекта)

На семинаре способы только перечислены в ДЗ, ниже по [документации Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/logging/). Рантайм пишет stdout и stderr контейнеров в файлы `/var/log/pods` на ноде.

| Способ                    | Плюсы                                      | Минусы                                       |
| ------------------------- | ------------------------------------------ | -------------------------------------------- |
| агент на ноде (DaemonSet) | один агент на ноду, приложение не меняется | -                                            |
| sidecar с агентом в поде  | свой конфиг под приложение                 | агент в каждом поде, `kubectl logs` не видит |
| push из приложения        | -                                          | приложение знает о хранилище, вне Kubernetes |

| Событие                  | Что с логами                                                   |
| ------------------------ | -------------------------------------------------------------- |
| контейнер перезапустился | остаются логи одного предыдущего: `kubectl logs --previous`    |
| под удалён или вытеснен  | удаляются с ноды вместе с контейнерами                         |
| файл вырос               | ротация: до 5 файлов по 10 MiB, `kubectl logs` видит последний |

Отсюда вывод документации: логам нужно хранилище, которое живёт независимо от подов и нод.

## Хранилища

| Тип                   | Примеры                   | Плюсы                        | Минусы                            |
| --------------------- | ------------------------- | ---------------------------- | --------------------------------- |
| полнотекстовый индекс | Elasticsearch, OpenSearch | мощный поиск с ранжированием | тяжёлый, индекс больше данных     |
| индекс по лейблам     | Loki                      | дёшево: сжатые чанки в S3    | поиск по тексту - полный скан     |
| колоночное            | ClickHouse, Victoria Logs | компактно, быстрые агрегации | нет ранжирования по релевантности |

([32:18](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=1938))

Что выбрать: ([40:11](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=2411))
- 30 сервисов в k8s, хранить год, бюджет минимальный, ищут по сервису и поду -> Loki: сервис и под и есть лейблы
- поддержка ищет по тексту обращений с ранжированием -> Elasticsearch
- error rate и p95 по 500 GB логов за неделю -> ClickHouse или Victoria Logs

**ELK** - Elasticsearch (хранение, поиск) + Logstash (обработка) + Kibana (UI). Легко начать, тяжело эксплуатировать на объёмах. ([37:14](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=2234))

## Victoria Logs

**Victoria Logs** - колоночная БД для логов от VictoriaMetrics. ([41:37](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=2497))
- один бинарник, Apache 2.0
- принимает протоколы Elasticsearch и Loki: шиппер менять не нужно
- встроенный UI vmui на `/vmui/`
- партиции по дням, сжатие 10x+, по умолчанию хранит 7 дней
- по семинаристу, логи в Т-Банке на стеке VictoriaMetrics ([57:04](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=3424))

| Поле      | Что это                                                 | Если нет в записи    |
| --------- | ------------------------------------------------------- | -------------------- |
| `_msg`    | текст; по нему ищет фильтр без имени поля               | обязательно          |
| `_time`   | время события                                           | время приёма         |
| `_stream` | поля-метки, как лейблы Loki; стрим лежит на диске рядом | дефолтный стрим `{}` |

([46:37](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=2797))
- формат логов приложения не меняют: маппинг полей задают в сборщике (`[sinks.vlogs.query]` выше)
- запрос начинают с `_time:`, иначе просматривается весь срок хранения

### LogsQL

([49:08](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=2948))

```
error                                   # слово в _msg
_time:5m level:error -(buggy OR foo)    # окно, поле, исключение
_time:5m level:error
  | fields _time, _stream, _msg         # пайпы, как в Unix
  | sort by (_time) desc
  | limit 10

# error rate по сервису - готовое условие алерта
_time:5m
  | stats by (service.name) count() logs, count() if (level:error) errors
  | math (errors / logs) as error_rate
  | filter error_rate:>0.1

_time:1h | top 5 by (method)                                # топ значений
_time:5m | stats by (method) quantile(0.95, duration_ms) p95
_time:5m status:>=500 | stats count_uniq(host) clients
```

Алерты: Victoria Logs -> vmalert (выполняет запрос по расписанию) -> Alertmanager (звонки, Telegram). Для панелей есть Grafana-плагин. ([54:20](https://tbank.ktalk.ru/recordings/1uK8BHn4CGk71vj1Vaxw?t=3260))

> [!note]- Пропущено
> подробности трёх сбоев, упражнения на поиск проблем и утечек, syslog, OpenSearch, Splunk, Graylog, почему не Postgres и Redis, бенчмарк и ограничения Victoria Logs, демо vmui, внутренняя платформа Т-Банка, ДЗ.
