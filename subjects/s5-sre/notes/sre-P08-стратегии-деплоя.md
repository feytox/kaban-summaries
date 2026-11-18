---
date: 2026-11-13
ingest: fast
links:
  - https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view
  - https://gateway-api.sigs.k8s.io/
  - https://gateway.envoyproxy.io/docs/
  - https://habr.com/ru/companies/flant/articles/471620/
  - https://blog.kubecost.com/blog/kubernetes-deployment-strategy/
  - https://github.com/ContainerSolutions/k8s-deployment-strategies
  - https://youtu.be/1oPhfKye5Pg
---
# P08. Стратегии деплоя

## Скорость против безопасности

Частый деплой быстрее приносит фичи и обратную связь, а маленькое изменение проще откатить и расследовать. Но с ним выше риск притащить баг на прод, поэтому нужны тесты, наблюдаемость и быстрый откат. ([03:30](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=3m30s))

**Стратегия деплоя** - способ выкатывать часто и при этом не ронять прод и откатываться за секунды. ([04:15](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=4m15s))

Прежде стратегии выбирают индикаторы влияния, по которым её оценят: ([06:48](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=6m48s))
- качественные: нулевой простой, тест на реальных пользователях, выбор целевой аудитории
- количественные: стоимость (ресурсы), длительность отката, вред пользователям, сложность настройки

## Gateway API вместо Ingress

По семинаристу, Ingress устарел и разработчики Kubernetes прекращают его поддержку. По документации API Ingress заморожен: его не развивают, но удалять не планируют, а для нового рекомендуют Gateway API (вне конспекта, [документация](https://kubernetes.io/docs/concepts/services-networking/ingress/)). ([12:12](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=12m12s))

Ingress описывает пару "хост + путь", а гибкую маршрутизацию каждый контроллер делает своими аннотациями. В Gateway API веса, матчи по заголовкам и зеркалирование есть из коробки и переносимы; по семинаристу, под высокой нагрузкой он медленнее Ingress. ([12:40](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=12m40s))

Gateway API делит ответственность по ролям, и стратегии деплоя меняют только HTTPRoute (см. также [[sre-P02-основы-kubernetes#Сеть|P02]]): ([13:58](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=13m58s))

```text
GatewayClass   каким контроллером обслуживается   - провайдер (здесь Envoy Gateway)
  Gateway      точка входа: адрес, порты, TLS     - администратор кластера
  HTTPRoute    куда и какой трафик идёт           - разработчик приложения
  backendRefs  Services с весами -> поды разных версий
```

Трафик HTTPRoute дробит тремя механизмами: ([16:17](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=16m17s))

| Механизм             | Что делает                                 | Стратегия          |
| -------------------- | ------------------------------------------ | ------------------ |
| weight               | доля трафика на каждый backend             | Canary, Blue/Green |
| match                | backend по заголовку, query, методу, пути  | A/B                |
| filter RequestMirror | копия запроса на другой backend            | Shadow             |

## Rolling и Recreate: один Deployment

Эти стратегии трафик не дробят: Kubernetes меняет поды одного Deployment по очереди или разом. ([24:44](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=24m44s))

```yaml
spec:
  replicas: 6
  strategy:
    type: RollingUpdate     # по умолчанию в Kubernetes
    rollingUpdate:
      maxSurge: 1           # сначала поднять лишний под: стабильность важнее ресурсов
      maxUnavailable: 0     # не гасить старый, пока новый не Ready
# maxSurge: 0 - сначала гасят старый под; экономит ресурсы, но при одном поде трафик оборвётся
# kubectl rollout undo deploy/my-app - откат, такая же постепенная раскатка
```

По семинаристу, новый под считается запущенным, когда прошли liveness- и readiness-пробы. По документации он доступен, когда Ready и продержался `minReadySeconds`, а Ready решает readiness-проба (вне конспекта, [документация](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)). ([25:19](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=25m19s))

**Rolling update** - поды новой версии постепенно заменяют старые без простоя. ([29:28](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=29m28s))
- во время раскатки v1 и v2 работают одновременно: нужна [[sre-L05-монолиты-и-микросервисы#Совместимость версий|совместимость версий]]
- подходит stateful-приложениям с ребалансировкой
	- но хранить состояние в Kubernetes семинарист категорически не советует (см. [[sre-P02-основы-kubernetes#Хранение|P02]])

**Recreate** - сначала удалить все поды, потом поднять новую версию. Его берут, когда одновременная работа v1 и v2 недопустима: несовместимые схемы данных, общий эксклюзивный ресурс, вендорская "коробка" с миграцией без отката. ([32:11](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=32m11s))
- простой длится время остановки плюс запуска, а с миграцией намного дольше
- откат тоже с простоем

## Blue/Green и Canary: две версии, веса

Обе версии подняты отдельными Deployment и Service, а трафик между ними переключает только HTTPRoute. ([36:01](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=36m1s))

```yaml
# canary: вес v2 меняют 10 -> 50 -> 100 по мере уверенности, откат - обнулить вес
rules:
  - backendRefs:
      - name: my-app-v1
        port: 80
        weight: 90
      - name: my-app-v2
        port: 80
        weight: 10
# blue/green: в маршруте один backendRef; apply маршрута на my-app-v2 переключает всех разом,
# apply маршрута на my-app-v1 - мгновенный откат
```

**Blue/Green** - новую версию целиком разворачивают рядом со старой, выполняют миграции и разом переключают на неё весь трафик. Старая простаивает, готовая к мгновенному откату. ([36:14](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=36m14s))
- цена - x2 ресурсов, большую часть времени половина простаивает
- баг новой версии заденет сразу всех
- у stateful-приложений перенести состояние между копиями нетривиально

Миграция БД blue/green не мешает, если у каждой версии своя база. Перед переключением прогревают кэши: иначе резко вырастет нагрузка на базу. ([40:59](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=40m59s))

**Canary** - новая версия получает малую долю реального трафика; по метрикам ошибок и латентности долю увеличивают или обнуляют. От rolling её отличает управляемое промежуточное состояние: долю можно держать на 10% сколько нужно. ([44:44](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=44m44s))
- на новую версию попадает случайный пользователь, выбрать кого нельзя
- нужна наблюдаемость, чтобы вовремя заметить деградацию
- раскатка медленная: вес поднимают шагами

Версии работают одновременно, поэтому миграция БД при канарейке недопустима: пользователи новой версии не заполнят поля старой, и при откате эти данные потеряются. ([18:22](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=18m22s))

По семинаристу, канарейка не требует лишнего железа, а в таблице на слайде её стоимость "средняя" [?] источники расходятся. Откат на слайде быстрый, а в итогах семинарист назвал его очень долгим [?] причину не пояснил. ([01:06:34](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=66m34s))

## A/B и Shadow: точная аудитория и копия трафика

**A/B testing** - канарейка, где аудиторию выбирают точно: по заголовку, cookie, query-параметру, гео. Так фичу дают сотрудникам банка или пользователям Safari; по слайду это скорее инструмент продуктового решения, чем стратегия деплоя. ([54:17](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=54m17s))

```yaml
rules:
  - matches:
      - headers:
          - type: Exact
            name: x-user-group
            value: beta
    backendRefs:
      - name: my-app-v2   # группа beta получает новую фичу
        port: 80
  - backendRefs:
      - name: my-app-v1   # все остальные - как было
        port: 80
```

**Shadow** - пользователь получает ответ старой версии, а копия его запроса уходит на новую, и её ответ отбрасывается. Новую версию проверяют органическим трафиком без риска для пользователей, но обе версии нагружены одновременно. ([59:39](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=59m39s))

```yaml
rules:
  - backendRefs:
      - name: my-app-v1        # отвечает пользователю
        port: 80
    filters:
      - type: RequestMirror
        requestMirror:
          backendRef:
            name: my-app-v2    # получает копию, ответ отбрасывается
            port: 80
```

## Какую стратегию брать

Сравнение со слайда: ([01:04:43](https://drive.google.com/file/d/1TP8j7MTF4I7OaajOszh52wu3h6UeurTV/view?t=64m43s))

| Стратегия  | Простой | Стоимость  | Скорость отката | Контроль трафика | Сложность |
| ---------- | ------- | ---------- | --------------- | ---------------- | --------- |
| Recreate   | есть    | низкая     | медленно        | нет              | низкая    |
| Rolling    | нет     | низкая     | медленно        | нет              | низкая    |
| Blue/Green | нет     | x2 ресурсы | мгновенно       | всё или ничего   | средняя   |
| Canary     | нет     | средняя    | быстро          | доля (вес)       | средняя   |
| A/B        | нет     | средняя    | быстро          | точная аудитория | высокая   |
| Shadow     | нет     | x2 ресурсы | -               | копия трафика    | высокая   |

- Rolling по семинаристу берут в 90% случаев за простоту
- Канарейку многие считают серебряной пулей, её сложность по семинаристу на деле невысокая
- A/B держит параллельно несколько версий для разных групп, например по браузерам
- A/B требует умного балансировщика и трассировки
- Shadow - не настоящий пользовательский тест
- Shadow иногда требует mocking

> [!note]- Пропущено
> байка про ночные деплои админов, feature flags и trunk-based development (по семинаристу - методы разработки, а не стратегии деплоя), опросы в чате, демо на minikube (скрипты, дашборд RPS по версиям, команды запуска), лаг мониторинга из-за scrape interval, тестовое приложение, ДЗ.
