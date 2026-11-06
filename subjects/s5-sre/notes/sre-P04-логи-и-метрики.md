---
date: 2026-10-09
ingest: full
links:
  - https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax
---
# P04. Логи и метрики для разработчиков

Код со слайдов, сокращён.

Метрики отвечают на вопрос "сломалось ли и насколько", логи - "почему сломалось". SRE живёт в метриках и спускается в логи только по алерту. ([05:49](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=349))

Демо-стенд: ([09:07](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=547))

```
orders-service --stdout JSON--> Vector --> VictoriaLogs          # логи, push
orders-service <--scrape 15s-- VictoriaMetrics                    # метрики, pull
VictoriaMetrics --> vmalert --> Alertmanager --> чат дежурного    # до человека доходят только алерты
VictoriaMetrics, VictoriaLogs --> Grafana
```

## Логи

Формат и поля - в [[sre-P03-логирование]].

Плохой лог: ([16:40](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=1000))

```
2026-10-04 12:03:11 ERROR Failed to create order 8812 for user ivan@mail.ru, amount=15000: java.n...
  at java.base/sun.nio.ch.NioSocketImpl.timedRead(...)
# поиск только regex'ом, а текст поменяют в следующем релизе
# стектрейс разваливается на десятки отдельных записей
# email в логе - утечка PII; время без часового пояса
```

### Уровни по семинаристу

([27:32](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=1652))

| Уровень | Когда                             | Пример                                           |
| ------- | --------------------------------- | ------------------------------------------------ |
| ERROR   | запрос не выполнен, нужен разбор  | авторизация не достучалась до хранилища секретов |
| WARN    | отклонение, но система справилась | пришёл `amount=-1`; пользователь ошибся паролем  |
| INFO    | бизнес-события, смена состояния   | заказ создан, перечитан конфиг                   |
| DEBUG   | детали для отладки                | на проде выключен                                |

- ERROR должен быть поводом что-то сделать: если на него никто не реагирует, это не ERROR
- по этому семинаристу, на проде пишут только ERROR, максимум WARN; семинарист P03 советовал `info` ([29:07](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=1747))

### Практики

- **trace_id** кладут в MDC на входе запроса и передают в исходящие вызовы ([32:04](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=1924))
	- он сквозной: часто его генерирует входной балансировщик, и сервисы его не перезаписывают
	- trace_id полезно показывать клиенту: поддержка передаёт его SRE, и те видят весь путь запроса ([36:05](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=2165))
- писать в stdout: ротация и доставка - задача платформы; файлы в поде забьют том ([33:36](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=2016))
- одну ошибку логирует только тот, кто обработал исключение, а не каждый слой стека ([34:33](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=2073))

```yaml
# Spring Boot 3.4+: JSON-логи без внешних библиотек
logging:
  structured:
    format:
      console: logstash          # или ecs
    json:
      stacktrace:
        max-throwable-depth: 20  # обрезать длинный стектрейс
```

```java
// trace_id из заголовка или новый; попадает в каждую запись лога
String traceId = Optional.ofNullable(req.getHeader("X-Request-Id"))
    .filter(h -> !h.isBlank()).orElseGet(TraceIdFilter::newTraceId);
MDC.put("trace_id", traceId);
res.setHeader("X-Request-Id", traceId);
try { chain.doFilter(req, res); } finally { MDC.remove("trace_id"); }
```

### Vector и VictoriaLogs

Обрабатывать логи дешевле в пайплайне до хранилища, чем хранить мусор и разбирать при поиске. ([37:14](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=2234))

```yaml
transforms:
  parse:
    type: remap
    inputs: [docker]
    source: |
      structured, err = parse_json(.message)
      if err == null && is_object(structured) {
        . = merge(., object!(structured))
      } else {
        .parse_error = true            # не JSON не теряем, а помечаем
      }
      .message = redact(string!(.message), filters: [r'[\w.+-]+@[\w-]+\.[\w.]+'])  # email -> [REDACTED]
```

```
_time:15m {service="orders-service"} level:ERROR      # ошибки сервиса за 15 минут
trace_id:"test-trace-42"                              # весь путь одного запроса
_time:1h level:ERROR | stats by (error_type) count() as errors | sort by (errors desc)
_time:1h parse_error:true                             # что не распарсилось
```

## Логи vs метрики

**Метрика** - заранее посчитанное число, состояние системы во времени (спидометр). **Лог** - история отдельных событий с контекстом (бортовой журнал). ([48:03](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=2883))

|                     | Логи                              | Метрики                                   |
| ------------------- | --------------------------------- | ----------------------------------------- |
| Что хранится        | каждое событие с контекстом       | агрегат: счётчик, значение, распределение |
| Объём при росте RPS | растёт линейно                    | почти не меняется                         |
| Хранение            | дорого, дни-недели                | дёшево, месяцы-годы                       |
| Потери              | сэмплируют, теряют при перегрузке | пропуск скрейпа не критичен               |
| Роль в инциденте    | диагностика                       | обнаружение и алертинг                    |

([49:23](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=2963))
- 10 сервисов x 200 RPS x 3 записи x 500 байт дают около 260 ГБ логов в сутки, метрики тех же сервисов - единицы МБ ([53:24](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=3204))

Почему не алертить по логам: ([55:44](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=3344))
- сообщение переименовали, и алерт молча перестал работать
- упавший сервис ничего не пишет, ошибок нет, алерт зелёный; метрика `up == 0` это поймает
- запрос по логам под нагрузкой медленный, а алерт проверяется каждые 15 с
- честный p99 из сэмплированных логов не получить

Связка: алерт по метрике показывает где и когда, логи в том же окне и по trace_id - почему. ([57:41](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=3461))

## Метрики

Приложение считает метрики и отдаёт текстом, хранилище забирает их по расписанию и ставит время точки. ([59:12](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=3552))

```
# TYPE http_server_requests_seconds histogram
http_server_requests_seconds_bucket{method="POST",status="200",uri="/api/orders",le="0.1"} 1520
http_server_requests_seconds_count{method="POST",status="200",uri="/api/orders"} 1904
# имя{лейблы} значение; временной ряд = имя + уникальный набор лейблов
```

| Тип       | Что это                                | Как читать                                |
| --------- | -------------------------------------- | ----------------------------------------- |
| Counter   | только растёт, обнуляется при рестарте | через `rate()` / `increase()`             |
| Gauge     | текущее значение, вверх и вниз         | как есть, `avg_over_time()`               |
| Histogram | распределение по бакетам               | `histogram_quantile()` по `rate(_bucket)` |

- имена в snake_case, в базовых единицах (`_seconds`, `_bytes`), счётчики с `_total` ([01:00:41](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=3641))
- у лейбла ограниченный набор значений: `method`, `status`, `uri` как шаблон; никогда ID, email или полный URL
- 5 uri x 4 status x 2 method x 15 бакетов x 3 инстанса = 1 800 рядов; с `user_id` на 100 000 пользователей - 180 млн, хранилище ляжет
- уникальное для события пишут в поле лога, описывающее класс событий - в лейбл метрики ([01:02:20](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=3740))

**Бизнес-метрики** - то, за чем следит бизнес: заказы в минуту, время открытия страницы. SRE алертит и по ним: технические метрики могут быть в норме, когда пользователь не получает данные за 2 секунды. ([01:04:32](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=3872))

```java
// Micrometer; не "orders.created": суффикс _created зарезервирован в OpenMetrics и будет отрезан
Counter.builder("orders.placed").register(registry);                       // orders_placed_total
Counter.builder("orders.failed").tag("reason", "payment").register(registry); // ограниченный набор значений
Timer.builder("orders.payment.duration").publishPercentileHistogram().register(registry);
```

```yaml
# vm/scrape.yml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: orders-service
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["orders-service:8080"]
```

```
sum by (uri) (rate(http_server_requests_seconds_count{application="orders-service"}[1m]))   # RPS
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m]))                                        # доля 5xx
histogram_quantile(0.99, sum by (le) (rate(http_server_requests_seconds_bucket[5m])))        # p99
up{job="orders-service"}                                                                     # жив ли
```

## Дашборды

Хороший дашборд за 5 секунд отвечает, здоров ли сервис. Методологию (RED, USE, 4 golden signals, см. [[sre-P05-sli-slo-sla]]) выбирают под задачу сервиса. ([01:09:06](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=4146))

```
1. Здоровье   up | доступность 30д vs SLO | доля 5xx сейчас   <- видит пользователь
2. RED        RPS по uri | доля 5xx | p50/p95/p99
3. Бизнес     заказы в минуту, ошибки оплаты
4. Ресурсы    heap | GC pause | потоки Tomcat                 <- причины
5. Логи       ошибки из VictoriaLogs в том же окне времени     <- детали
```

- дашборды и источники данных хранят как код (JSON в репозитории + provisioning), чтобы быстро восстановить мониторинг после сбоя ([01:13:01](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=4381))
- не больше 15-20 панелей; пороги те же, что в алертах; из алерта ссылка на дашборд, из дашборда на runbook ([01:15:08](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=4508))

## Пороги и алерты

Алерт будит человека, только когда страдают пользователи и нужно действие. Механика алертов в [[sre-P06-алерты]]. ([01:16:23](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=4583))
- алертить по симптомам, а не по причинам: 5xx у пользователей - алерт, CPU 85% - панель, максимум warning ([01:17:31](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=4651))
- порог берут из истории: p99 за 2-4 недели, выше нормального пика, но ниже того, что заметит пользователь

**Burn rate** - во сколько раз быстрее нормы сгорает бюджет ошибок.

| Алерт              | Окна         | Burn rate | Порог 5xx при SLO 99,5% | Severity                 |
| ------------------ | ------------ | --------- | ----------------------- | ------------------------ |
| быстрое сжигание   | 1 ч и 5 мин  | 14,4      | 7,2%                    | critical, будит          |
| медленное сжигание | 6 ч и 30 мин | 6         | 3%                      | warning, в рабочее время |
| сервис недоступен  | 1 мин        | -         | `up == 0`               | critical                 |

Длинное окно подтверждает, что проблема значимая, короткое - что она ещё идёт: с одним часовым окном алерт горит час после починки. ([01:18:43](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=4723))

```yaml
- alert: OrdersErrorBudgetFastBurn
  expr: |
    job:http_error_ratio:rate1h{job="orders-service"} > (14.4 * 0.005)
    and job:http_error_ratio:rate5m{job="orders-service"} > (14.4 * 0.005)
  for: 2m
  labels: { severity: critical }
  annotations:
    runbook_url: "https://wiki/runbooks/orders-service#errors"
    dashboard: "http://localhost:3000/d/orders"
# job:http_error_ratio:rate1h - recording rule: доля 5xx за час, посчитанная заранее
```

## Инцидент и что SRE ждёт от разработчика

Цикл реагирования: ack, масштаб по метрикам, митигация (откат, рестарт, фича-флаг), диагностика по логам, подтверждение по метрикам, постмортем. Сначала восстанавливают сервис, потом ищут причину. ([01:20:22](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=4822))

От разработчика SRE ждёт: ([01:21:31](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=4891))
- JSON-логи с trace_id, RED и бизнес-метрики без высокой кардинальности
- дашборд и SLO-алерты в репозитории рядом с кодом
- **runbook** на каждый critical-алерт: как быстро восстановить сервис; поначалу простой, вроде "завис - `systemctl restart` или удалить под" ([01:24:11](https://tbank.ktalk.ru/recordings/kCRylgjeilgatIpvE9ax?t=5051))

> [!note]- Пропущено
> демо vmui и дашборда (на семинаре не заработало), live-инцидент и его хронология, пример runbook, чек-лист готовности сервиса (повторяет заметку), жизненный цикл алерта и маршрутизация Alertmanager (подробно в P06), настройка Micrometer в `application.yml`, практика по желанию, ДЗ.
