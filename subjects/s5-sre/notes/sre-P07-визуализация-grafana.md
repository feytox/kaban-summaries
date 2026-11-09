---
date: 2026-11-06
ingest: full
links:
  - https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt
  - https://grafana.com/docs/
  - https://play.grafana.org
  - https://habr.com/ru/companies/qiwi/articles/673228/
---
# P07. Визуализация: Grafana

Код из демо восстановлен по речи, имена сервисов и метрик мои.

Алерт сообщает, что что-то сломалось, а понять, что происходит с сервисом, помогает дашборд: несколько понятных графиков на одном экране. Без нормальной визуализации решение принять нельзя. ([02:49](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=169))

## Хороший дашборд

Эталон - приборная панель автомобиля: на ней всё нужное для решения и ничего лишнего. ([05:30](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=330))

- **не избыточен** и **информативен**: только важные метрики, но все
- **быстро изучается** и **компактен**: состояние видно с одного взгляда

## Почему Grafana

| Инструмент      | Что это               | Почему не для мониторинга     |
| --------------- | --------------------- | ----------------------------- |
| Kibana          | экосистема Elastic    | в основном визуализация логов |
| Tableau         | BI, сложные отчёты    | аналитика, не оперативка      |
| Apache Superset | open-source BI        | то же                         |
| Excel           | универсальная таблица | то же                         |

([06:59](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=419))

Grafana - фактический стандарт: низкий порог входа и мощные инструменты для SRE. От остальных её отличают обновление в реальном времени (интересно состояние сейчас, а не час назад) и интерактивность: фильтры, переменные, выбор периода. ([09:57](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=597))

## Типы визуализаций

**Панель** - окно с одним графиком. **Дашборд** - одна или несколько панелей, логически описывающих состояние чего-то: сервиса, Postgres. ([11:24](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=684))

| Тип         | Что показывает                           | Для чего                                           |
| ----------- | ---------------------------------------- | -------------------------------------------------- |
| Stat        | одно крупное значение, текущее или сумма | жив ли сервис, RPS, ошибки сейчас; верх drill-down |
| Gauge       | значение на шкале с цветными порогами    | насколько далеко от нормы, без чтения цифр         |
| Bar Gauge   | несколько значений-столбиков с порогами  | сравнить сущности, найти проблемную                |
| Time Series | зависимость значения от времени          | по умолчанию: RPS, задержки, ошибки, утилизация    |
| Heatmap     | распределение по времени и величине      | задержки по бакетам Histogram                      |
| Statusmap   | состояние множества объектов во времени  | строка - объект (под), цвет - состояние            |
| Polystat    | много значений-плиток с цветом           | панель состояния набора сервисов                   |
| Pie Chart   | доли от целого                           | ошибки по статус-кодам, запросы по методам         |

- пороги раскрашивают шкалу: зелёный - норма, жёлтый - на грани, красный - критично ([13:47](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=827))
- запоминать все типы не нужно, важно знать, какой тип для какой задачи ([16:43](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1003))

**Drill-down** - сначала обобщённая картина (доля ошибок, норма ли CPU), затем детализация шаг за шагом до инстанса и запроса. Цель - пройти по пути клиента и быстро найти шаг, где ошибки. Дашборды из [[sre-P04-логи-и-метрики#Дашборды]] - тот же подход; по Мусину, Тимофеев называл их варианты "Топдаун, бот, мап" [?] в P04 этих названий нет. ([17:36](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1056))

## Настройка панели

([19:21](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1161))

- **единица измерения** обязательна: Grafana переводит unix timestamp в часы и дни, байты в КБ, считает проценты
- **ссылки** на документацию и runbook прямо с панели
- **часовой пояс**: по умолчанию системный пояс компьютера, его можно задать жёстко
- **сетка** и **логарифмическая шкала**: лог-шкала удобна, когда числа большие и важен порядок
- стиль графика ([21:50](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1310)): линии, столбцы или точки, сглаживание, толщина, прозрачность, заполнение пропусков (connect null values)
- **thresholds** - линии порогов, например 80% для утилизации CPU

## Инструменты дашборда

**Переменная** - значение вместо хардкода в запросах; ею фильтруют панели. Разделение по системам (БД, балансеры, приложения), окружениям, продуктам и узлам есть почти в любом реальном дашборде; делить сами дашборды по окружениям неудобно. ([23:02](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1382))

**Аннотация** - метка события на графике, ставится вручную: деплой, смена лимитов, старт рекламной кампании, Чёрная пятница. Через месяцы она объясняет скачок метрики и вам, и коллегам. ([23:55](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1435))

- **алерты** в Grafana есть (вкладка Alert -> New alert rule, условие, аналог `for`, contact points), но семинарист не встречал, чтобы ими пользовались: стандарт - Alertmanager, см. [[sre-P06-алерты#Алерты в Grafana]] ([25:34](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1534))
- **теги** помогают искать среди сотен дашбордов, на визуализацию не влияют ([27:16](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1636))
- **строки (rows)** группируют панели по смыслу (SLI/SLO, ресурсы, бизнес-метрики) и сворачиваются одним щелчком ([28:00](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1680))
- **повторение (Repeat for)**: панель или строку рисуют один раз, и Grafana размножает её по значениям переменной, например `env`: dev, test, prod ([28:44](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1724))
- **ссылки** на другие дашборды или любые URL - тоже drill-down: с общего дашборда продукта переходят на дашборд "красного" сервиса ([29:37](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1777))

## Dashboard as Code

Любой дашборд выгружается в JSON, и есть библиотеки, которые генерируют этот JSON кодом: его можно версионировать в git. На слайде пример на Kotlin DSL, устно упомянута библиотека для Python [?] название в записи неразборчиво. ([30:52](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=1852))

- по Мусину, нужно редко: единого стандарта нет, каждая команда выбирает свой инструмент, сам он не пользовался; полезно для версионирования и восстановления после сбоя Grafana
- Тимофеев в [[sre-P04-логи-и-метрики#Дашборды]], наоборот, советовал хранить дашборды как код

## Демо

Grafana ставится в namespace мониторинга Helm-чартом ([[devops-L07-ci-cd-git-helm#Helm]]); ревизии релиза Helm хранит в Secret'ах, поэтому умеет откатываться. ([36:37](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=2197))

```bash
helm install grafana grafana/grafana -n monitoring -f grafana-values.yaml
helm list -n monitoring        # что установлено
helm rollback grafana          # откат на прошлую ревизию, если в релизе нашли баг
```

**Data source** - абстракция источника данных: через него Grafana обращается к TSDB или СУБД (Postgres, MySQL, ClickHouse...). ([39:10](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=2350))

```yaml
# grafana-values.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: VictoriaMetrics
        type: prometheus          # формат метрик VM совместим с Prometheus
        url: http://victoria-metrics.monitoring.svc:8428
        isDefault: true           # подставляется в новые панели
```

- по семинаристу, логин и пароль по умолчанию `admin`/`admin`, если не заданы в values ([41:38](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=2498)); Helm-чарт без `adminPassword` генерирует случайный пароль в Secret (вне конспекта, [README чарта](https://github.com/grafana-community/helm-charts/blob/main/charts/grafana/README.md))
- **Explore** строит график по запросу без дашборда, например `pg_up` ([42:44](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=2564))
- новый дашборд: Add visualization -> data source -> запрос -> тип панели -> Save в папку ([44:11](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=2651))
- под типовые системы (Postgres, Redis, Spring Boot Actuator, Django) рисовать свои дашборды незачем: готовый ищут на grafana.com по экспортеру, скачивают JSON и импортируют через New -> Import ([45:25](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=2725))

```
label_values(pg_up, instance)   # переменная instance: значения лейбла из метрики
pg_up{instance="$instance"}     # переменная в запросе панели
```

По семинаристу, `label_values` встроена в PromQL ([48:10](https://tbank.ktalk.ru/recordings/8tJQcEX253EiTjSYWREt?t=2890)); по документации это синтаксис переменных Grafana (Classic query, устаревший), а не PromQL (вне конспекта, [docs](https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/)).

> [!note]- Пропущено
> трейдеры у мониторов, история слайда с панелью Volkswagen, Grafana Loki и блог Grafana Labs, вкладка администрирования и способы авторизации, Doom в Grafana, ДЗ.
