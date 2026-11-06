---
date: 2026-10-23
ingest: full
links:
  - https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU
---
# P06. Алерты

Конфиги со слайдов и демо, сокращены; конфиг Alertmanager собран из нескольких слайдов.

Метрики и дашборды не помогут, если на них никто не смотрит: алерт сообщает о проблеме в мессенджер, SMS или рабочий чат, даже когда инженер не у компьютера. ([04:34](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=274))

## Архитектура

```
vmagent (сбор, remote_write) -> VictoriaMetrics (хранение, запросы) -> vmalert (оценка правил)
  -> Alertmanager (группировка, подавление, маршрутизация) -> email/SMS, Slack/Telegram, PagerDuty
VictoriaMetrics -> Grafana (дашборды)
```

- Prometheus умеет всё это одним бинарником: скрейпит, хранит, проверяет правила ([01:11:05](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=4265))
- стек VictoriaMetrics разбит на компоненты, и каждый можно масштабировать и реплицировать; курс выбрал его как более актуальный
- vmagent тоже реплицируют: если единственный сборщик упадёт, загорятся все алерты, хотя сервисы живы ([01:11:51](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=4311))
- хранилище метрик - stateful-приложение, в Kubernetes его обслуживать неудобно и в облаке дорого; проще вынести на виртуалку или взять Mimir с данными в S3 ([01:14:49](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=4489))

## PromQL

**PromQL** - язык запросов к time series БД, не SQL-подобный. Временной ряд - имя метрики и набор лейблов, по которым ряды различают. ([06:42](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=402))

```
http_requests_total{environment=~"qa|dev-.*", method!="GET"}   # селекторы: = != =~ !~
{job="oncall"}                                                  # без имени: имя - служебный лейбл __name__
node_filesystem_bytes_avail{mountpoint="/", environment="production"} / 1e9 < 5   # меньше 5 ГБ свободно
sum(http_requests_total)                                        # агрегация: sum, min, max, avg
sum by (namespace) (kube_pod_container_resource_requests)       # сумма по значениям лейбла
rate(http_requests_total{job="api-server"}[5m])                 # скорость роста counter за окно
```

- график Counter без `rate` неудобен: значение только растёт и обнуляется при рестарте ([18:15](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=1095))
- у VictoriaMetrics набор функций немного отличается от Prometheus, смотреть её документацию ([15:31](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=931))
- запросы в алертах бывают огромными: писать их можно с ИИ, но читать нужно уметь самому ([01:08:29](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=4109))

## Алерты в Grafana

Grafana умеет алертить сама: настройка из UI, всё из коробки. Подходит для случая "хочу приглядывать за...", но надёжность ниже, а хранить такие алерты в git тяжело. В ДЗ алерты из Grafana не принимают. ([25:10](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=1510))

## vmalert

vmalert выполняет запросы правил по расписанию и передаёт сработавшие алерты в Alertmanager. ([28:22](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=1702))

```bash
# откуда брать метрики, куда слать алерты, где лежат правила
./vmalert \
  -datasource.url=http://victoriametrics:8428 \
  -notifier.url=http://alertmanager:9093 \
  -rule="/etc/vmalert/alerts/*.yml"
```

```yaml
groups:
  - name: example                  # группы - по сервисам или проектам
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status="500"}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.1
        for: 5m                    # порог должен держаться 5 мин: отсекает скачки
        labels:
          severity: warning
        annotations:
          summary: "Доля ошибок 5xx выше 10%"
          runbook: "https://wiki/..."    # как реагировать
```

- **severity** ставят на каждый алерт, по нему выбирают канал: critical - SMS дежурному ночью, warning - посмотреть утром ([32:43](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=1963))
- в аннотации кладут runbook и ссылку на дашборд Grafana

**Состояния алерта**: inactive - условие ложно; pending - условие истинно меньше `for`; firing - истинно дольше, алерт уходит в Alertmanager. Когда условие снова ложно, Alertmanager шлёт resolved без задержки. ([51:46](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=3106))

## Alertmanager

**Alertmanager** получает уже сработавшие алерты и решает, кому, как часто и в какой форме о них сообщить: группирует, глушит, маршрутизирует по каналам и может дублировать в несколько. ([26:39](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=1599))

Конфиг состоит из получателей (`receivers`) и маршрутов (`route`); нужен хотя бы один маршрут. ([33:29](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=2009))

```yaml
route:
  receiver: home_email             # по умолчанию: сюда всё, что не попало в routes
  group_by: [alertname]            # алерты с одинаковым alertname - одно уведомление
  group_wait: 30s                  # ждать 30 с, собирая группу
  group_interval: 5m               # новые алерты в группе - не чаще раза в 5 мин
  repeat_interval: 12h             # напоминать о горящем алерте
  routes:
    - matchers:                    # все условия должны выполниться
        - group = "oncall"
        - env =~ "prod|test"
        - severity = "critical"
      receiver: oncall-critical
      continue: false              # false: дальше не проверять; true: проверить и следующие маршруты
receivers:
  - name: oncall-critical
    email_configs:
      - to: critical-monitoring@somesite.ru
        send_resolved: true
    slack_configs:
      - channel: '#critical-alerts'
        http_config:
          proxy_url: http://proxy.somesite.ru:8080/   # в примере Slack доступен только через прокси
    webhook_configs:
      - url: http://sms.somesite.ru/alert?phones=79031234567   # SMS через свой сервис
```

- в новых версиях `matchers` один список для обычных условий и regex; раньше были `match` и `match_re` ([41:12](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=2472))
- семинарист на практике группировал только по `alertname`
- последним маршрутом стоит завести получатель-"чёрную дыру", чтобы видеть алерты, не попавшие ни в один маршрут ([56:46](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=3406))
- текст уведомления задают Go-шаблоном из лейблов и аннотаций; для Telegram шаблон проще, ссылки там нажимать неудобно ([44:06](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=2646))
- Mattermost принимает Slack-совместимые вебхуки

**Silence** - временное глушение алертов по лейблам, например на время плановых работ. Задают срок, автора и комментарий: иначе через месяц никто не знает, чей это silence. Кнопка Preview показывает, какие алерты попадут под него. ([01:01:03](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=3663))

## Демо

vmalert поставлен в minikube оператором через Helm, правила описаны ресурсом `VMRule`. Алерт `OnCallDown` с `expr: up{job="oncall"} == 0` и `for: 1m` сработал после `kubectl scale --replicas=0`: сначала pending, через минуту firing, и в Mattermost пришли сообщения от Alertmanager и от Grafana. ([46:16](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=2776))

## Алерты по логам

По семинаристу, допустимы, но дороже: запросы по логам тяжелее. Kibana и Graylog так умеют. Если алерт по логам нужен постоянно, лучше доработать приложение и завести метрику; это невозможно только для чужого приложения (blackbox). Подробнее в [[sre-P04-логи-и-метрики#Логи vs метрики]]. ([01:04:53](https://tbank.ktalk.ru/recordings/4umVHCgLhoMuX7Lu1TcU?t=3893))

> [!note]- Пропущено
> демо Grafana Explore и функций агрегации, пример каскадной маршрутизации со слайда, Elastic и Zabbix в ответах на вопросы, ДЗ.
