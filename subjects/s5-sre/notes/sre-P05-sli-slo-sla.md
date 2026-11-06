---
date: 2026-10-16
ingest: full
links:
  - https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p
---
# P05. Проектирование метрик, SLI/SLO/SLA

Код со слайдов, сокращён.

## Подходы к выбору метрик

([05:01](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=301))

| Подход         | Для чего            | Сигналы                                                                            |
| -------------- | ------------------- | ---------------------------------------------------------------------------------- |
| RED            | сервисы с запросами | Rate (RPS), Errors (запросы с ошибкой), Duration (распределение времени обработки) |
| USE            | ресурсы             | Usage (сколько времени ресурс занят), Saturation (загруженность, очередь), Errors  |
| Golden Signals | универсальный       | Latency, Traffic (востребованность сервиса), Errors, Saturation                    |

- подходы - рекомендации, а не правила: если сервис страдает от загрузки CPU, лучше взять USE ([01:08:55](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=4135))
- смотреть нужно путь запроса пользователя через все сервисы, БД и очереди, а не одну метрику; низкоуровневые CPU, память, сеть тоже полезны ([09:44](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=584))

## SLI, SLO, SLA

По семинаристу, подход годится для быстрых операций запрос-ответ. Лектор считает его устаревшим ([[sre-L01-надёжность-и-доступность]]), но он широко распространён. ([10:44](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=644))
- "надежда - не стратегия": иногда проще научиться быстро восстанавливаться, чем написать сверхнадёжный сервис
- в большой компании все должны считать доступность одинаково ([12:33](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=753))
- по куратору, платёж в банке проходит через десятки микросервисов; для доступности таких услуг в Т-Банке сделали отдельный инструмент ([01:13:48](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=4428))

**SLI** (Service Level Indicator) - индикатор пользовательского опыта: доля 5xx, p99 времени ответа, доступность в процентах. ([13:32](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=812))

**SLO** (Service Level Objective) - целевое значение SLI, например 5xx не более 5% запросов; считают за минуту, час, день. Целевой процент доступности - это девятки. ([15:10](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=910))

**SLA** (Service Level Agreement) - обещание бизнесу: SLO выполняется столько-то, а за невыполнение мы отвечаем. ([17:06](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=1026))
- 100% недостижимо, поэтому закладывают запас
- 99,999% - это 26 с простоя в месяц и 5 мин 13 с в год (калькулятор uptime.is) ([19:13](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=1153))
- лучше заявить меньше и наращивать, чем заявить много и весь квартал не выполнять

**Бюджет ошибок** - допустимый простой, который остаётся при заданном SLA. ([20:58](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=1258))

$$SLA(\%) = \frac{TWT - TDT}{TWT} \times 100\%$$

$TWT$ - всё время за период подсчёта, $TDT$ - время простоя.
- оставшийся бюджет можно тратить на плановые работы ([21:59](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=1319))
- если бюджет превышен, чинят надёжность вместо новых фич

### Алгоритм

([23:00](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=1380))
1. критичные пользовательские сценарии и бизнес-метрики для SLI
2. критичные сервисы, серверы и компоненты в этих сценариях
3. набор метрик, которые войдут в SLI
4. фактические значения выбранных SLI
5. SLO и бюджет ошибок
6. SLA с бизнесом

Сначала смотрят, как метрики ведут себя на деле, потом ставят цели и только затем обещают бизнесу.

## Blackbox, Graybox, Whitebox

Подходы к выбору SLI по глубине доступа к системе. ([26:39](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=1599))

| Уровень  | Доступ                                            | Что видит                                                   |
| -------- | ------------------------------------------------- | ----------------------------------------------------------- |
| Blackbox | только как пользователь: HTTP, ping               | работает или нет                                            |
| Graybox  | метрики ОС, хостов, контейнеров, логи, компоненты | где плохо: сервер, БД, очередь; путь "API -> БД -> очередь" |
| Whitebox | код, метрики, логи и трейсы в приложении          | стек вызовов, медленный метод, бизнес-метрики               |

Проверки Blackbox: ([29:27](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=1767))
- HTTP: статус 200, тело ответа, время отклика, срок TLS-сертификата
- TCP: открыт ли порт (443, 5432); ICMP: ping, потеря пакетов
- DNS: резолвится ли домен и как быстро
- прокси и балансировщики: редиректы, 5xx
- доступность внешних API и зависимостей

Разница на примерах: ([32:38](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=1958))
- запрос вернул 500: Blackbox видит, что сайт сломался; Graybox - что запрос упал при вызове БД
- порт 443 открыт: Blackbox считает, что всё работает; Graybox видит CPU ноды 98% и растущую очередь, то есть сервис на грани отказа

На уровне Whitebox задача SRE - добиться от разработки метрик, логов и трейсов, которых не хватает для быстрой реакции. ([35:48](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=2148))

## Демо: метрики и пробер

Flask-приложение с методом `/get_method`: 13% ответов 500, 3% медленных (1,1-3 с). ([37:58](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=2278))

```python
request_count = Counter('http_requests_total', 'Total HTTP requests',
                        ['method', 'endpoint', 'status'])
request_duration = Histogram('http_request_duration_seconds', 'HTTP request duration',
                             ['method', 'endpoint'], buckets=[1, 2, 3])

@app.before_request
def start_timer():
    g.start_time = time.time()

@app.after_request
def record_metrics(response):           # метрики по фактическому статусу ответа
    duration = time.time() - g.start_time
    request_count.labels(method=request.method, endpoint=request.path,
                         status=response.status_code).inc()
    request_duration.labels(method=request.method, endpoint=request.path).observe(duration)
    return response
# /metrics отдаёт generate_latest(REGISTRY) в формате Prometheus
```

**Пробер** - инструмент или скрипт, который сам шлёт синтетические запросы и так проверяет доступность и работоспособность компонента. ([40:56](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=2456))
- о проблемах узнаём раньше пользователей
- знаем, что всё работает, даже когда трафика нет
- позволяет проводить работы на проде: под рискованный релиз пишут пробер и следят за ним; если что-то не так, сначала откатывают, потом разбираются ([42:58](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=2578))
- после крупного сбоя, например отключения ДЦ, проберы первыми дают сигнал

## Определение сбоя

([47:37](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=2857))

| Тип          | Метод                              | Примеры                                                                                   |
| ------------ | ---------------------------------- | ----------------------------------------------------------------------------------------- |
| статический  | прямой: чёткий критерий отклонения | 5xx, рост времени ответа, разрыв сети, падение сервиса, сервис перестал выполнять функцию |
| динамический | косвенный: детектирование аномалий | всплеск трафика ночью, пропажа трафика днём                                               |

Аномалию трудно зашить в метрику с порогом, поэтому её ловят анализаторами, в том числе ML-моделями. ([49:33](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=2973))

## Пример: SLI, SLO, SLA для /get_method

Сценарий - запрос к `/get_method`, критичный сервис - само приложение, технические метрики - время ответа и доля успешных ответов. ([51:20](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=3080))

На семинаре "SLA" - доля точек за последний час, в которых SLO выполнялся. ([56:21](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=3381)) (вне конспекта) В [Google SRE Book](https://sre.google/sre-book/service-level-objectives/) SLA - договор с пользователями с последствиями за невыполнение SLO, а такая доля - просто выполнение SLO.

| Метрика      | SLI               | SLO   | Заявленный SLA |
| ------------ | ----------------- | ----- | -------------- |
| время ответа | p95 за 5 мин      | < 1 с | 80-83%         |
| ошибки       | доля 5xx за 5 мин | < 0,2 | 78-87%         |

```
# SLI времени ответа
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{endpoint="/get_method"}[5m])) by (le))
# SLI ошибок
sum(rate(http_requests_total{endpoint="/get_method",status="500"}[5m]))
  / sum(rate(http_requests_total{endpoint="/get_method"}[5m]))
# "SLA": доля точек за час, где SLO выполнен
count_over_time((SLI < 1)[1h:]) / count_over_time(SLI[1h:])
# комбинированный: оба SLO выполнены одновременно
count_over_time((SLI1 < 1)[1h:] & (SLI2 < 0.2)[1h:]) / count_over_time(SLI1[1h:])
```

- SLA повышают постепенно: 80% за квартал выдержали, ставят 85%, потом 87%; провалы на графике - аргумент для разработчиков заняться надёжностью ([59:20](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=3560))
- у приложения должен быть один SLA: комбинированный ниже каждого отдельного, зато честнее; отдельным SLO можно дать веса ([01:04:52](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=3892))

## Когда RED не подходит

Команда семинариста доставляет данные в хранилище, и RPS с временем ответа у неё нет. Главная метрика - время доставки изменения из источника в хранилище: SLO "отставание не больше 3 часов", а SLA - доля потоков, которые сейчас его нарушают. ([01:09:59](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=4199)) ([01:12:35](https://tbank.ktalk.ru/recordings/gLPiGM1VFenyTLvm3a3p?t=4355))

Пропущено: определения метрики, код пробера, docker-compose и конфиги Prometheus и Grafana, анонс техтолка, ДЗ.
