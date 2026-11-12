---
date: 2026-10-16
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk
---
# L05. kubectl, Deployment, ресурсы, ConfigMap и Secret

Практика на локальном кластере kind: развернуть приложение в Kubernetes, задать ему ресурсы и передать настройки и секреты. Манифесты восстановлены по трансляции, команды частично мои.

## kubectl, контекст и неймспейс

`kubectl` общается с kube-apiserver кластера ([[devops-L04-системы-хостинга#Kubernetes держит желаемое состояние]]). С каким кластером и от чьего имени, задаёт контекст. ([12:51](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=771))

**Контекст** - запись в `~/.kube/config`: кластер (адрес и сертификат), пользователь и, необязательно, неймспейс по умолчанию. Сессии у `kubectl` нет.

**Неймспейс** - логическое разделение одного физического кластера, чтобы приложение одной команды не перетёрло одноимённое приложение другой. ([08:39](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=519))

```bash
kubectl config use-context kind-hostinglab
kubectl config set-context --current --namespace=hostinglab   # больше не писать -n и --context
kubectl get pods -A                    # поды во всех неймспейсах
kubectl get pods -l app=web            # поды по лейблу
kubectl apply -f deployment.yaml       # привести объект к манифесту
kubectl describe pod <имя>             # описание и события: scheduler, pull образа, запуск
kubectl logs -f <имя>                  # stdout приложения
kubectl exec -it <имя> -- bash         # зайти в контейнер, как docker exec
kubectl port-forward deployment/web 8081:8080   # проброс порта через API кластера
```

Port-forward живёт, пока запущена команда, и попадает в один случайный под. ([1:13:24](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4404))

## Requests и limits

Requests - нижняя граница: ресурсы, без которых приложение не живёт, по ним scheduler выбирает ноду. Limits - верхняя граница, при выходе за которую приложение тормозят или убивают. ([29:55](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=1795))

```yaml
resources:
  requests:
    cpu: 100m                 # 0.1 ядра; 1 = 1000m = одно ядро
    memory: 64Mi
    ephemeral-storage: 64Mi   # временное место на диске ноды
  limits:
    cpu: "1"                  # превысил - троттлинг через cgroups
    memory: 256Mi             # превысил - OOM killer
    ephemeral-storage: 256Mi
```

- Без requests берутся неизвестные умолчания неймспейса, поэтому лектор советует указывать их всегда. ([31:42](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=1902))
- Request не может быть больше limit: такой манифест API отклоняет. ([33:48](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2028))
- OOM killer убивает корневой процесс контейнера, а с ним и все дочерние; контейнер выходит с ненулевым кодом. ([32:19](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=1939))
- По лектору, за превышение `ephemeral-storage` тоже убивает OOM killer. По документации kubelet выселяет (evict) такой под (вне конспекта, [Ephemeral storage](https://kubernetes.io/docs/concepts/storage/ephemeral-storage/#resource-emphemeralstorage-consumption)).

## Deployment над ReplicaSet над подом

Под - самый неудобный способ держать реплики: его спеку нельзя поменять на ходу, только удалить и создать заново, а второй под с тем же именем не создать. Поэтому поды руками не пишут: ими управляет ReplicaSet, а им - Deployment. ([34:35](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2075))

| Объект     | Что описывает                          | Что делает                                         |
| ---------- | -------------------------------------- | -------------------------------------------------- |
| Pod        | контейнеры                             | запускает контейнеры на одной ноде                 |
| ReplicaSet | `replicas`, `selector`, шаблон пода    | держит нужное число подов с лейблом из селектора   |
| Deployment | то же плюс настройки выкатки           | создаёт ReplicaSet'ы и решает, как менять реплики  |

([37:41](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2261), [43:13](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2593))

**Лейбл** - метка `ключ: значение` на объекте; по ней ReplicaSet собирает свои поды, а `kubectl` фильтрует вывод. ([27:22](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=1642))

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web              # какие поды считать своими
  template:                 # шаблон пода
    metadata:
      labels:
        app: web            # должен совпасть с селектором
    spec:
      containers:
        - name: web
          image: localhost:5001/probes-web:v2   # без адреса registry образ ищут на Docker Hub
```

- Удалённый под ReplicaSet тут же пересоздаёт: кластер стремится держать число подов равным `replicas`. ([42:31](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2551))
- Изменение Deployment, скорее всего, перезапустит поды; с одной репликой это простой. ([58:47](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3527)) По документации выкатку запускает только изменение `.spec.template` (вне конспекта, [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)).
- При обновлении создаётся новый ReplicaSet, и реплики постепенно переезжают в него; старые ReplicaSet'ы остаются с 0 реплик. ([1:29:33](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=5373))
- `kubectl rollout restart deployment/web` перезапускает реплики; по одной - только когда настроены пробы. ([49:14](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2954))

## Настройки и секреты приходят снаружи образа

Настройки, зависящие от окружения, передают в под через ConfigMap и Secret, чаще всего как переменные окружения, а можно и файлами через тома. Зашивать настройки в Docker-образ - "путь террориста" ([[devops-L03-dive-хранение-compose#Один образ на все окружения]]). ([53:08](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3188), [1:00:15](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3615))

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_ENVIRONMENT: lab
  BACKEND_HOST: backend
  BACKEND_PORT: "8080"
---
apiVersion: v1
kind: Secret
metadata:
  name: web-credentials
data:
  BACKEND_USERNAME: YWRtaW4=     # base64, а не шифрование
  BACKEND_PASSWORD: c2VjcmV0
```

```yaml
# в контейнере Deployment
envFrom:
  - configMapRef:
      name: web-config          # все ключи сразу
env:
  - name: BACKEND_USERNAME
    valueFrom:
      secretKeyRef:             # один ключ из секрета
        name: web-credentials
        key: BACKEND_USERNAME
```

- Отдельный ключ берут, чтобы раздать, например, API-ключ, не раздавая пароль из того же секрета. ([1:08:20](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4100))

Secret технически почти не отличается от ConfigMap: значения лежат в base64 и читаются любым, у кого есть доступ. Он лишь не выводит значения в `describe` и позволяет дать разработчикам права на ConfigMap, но не на секреты. ([1:03:11](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3791))

- Base64 позволяет хранить значения со спецсимволами и многострочные, например приватный SSH-ключ, и класть их в контейнер файлом. ([1:04:40](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3880))

Изменённый ConfigMap или Secret запущенные поды не видят: значения подтягиваются только при перезапуске. Сам Kubernetes за этим не следит, поэтому нужна система деплоя или плагин, который перезапускает поды при изменении. ([1:10:37](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4237), [1:11:48](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4308))

- По лектору, в демо не помог даже `rollout restart`, пришлось передеплоить. [?] По документации переменные окружения из ConfigMap обновляются при перезапуске пода, так что `rollout restart` должен помогать (вне конспекта, [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/#using-configmaps-as-environment-variables)).

## Состояние кластера меняют только через Git

Ручной `kubectl scale` меняет кластер здесь и сейчас, но следующий прогон CI с `kubectl apply` вернёт всё как в манифесте. Поэтому все изменения идут через Git и CI: это подход GitOps. ([47:56](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2876))

- Манифесты лежат в Git рядом с кодом или в отдельном репозитории; в отделе Рубцова их меняют через merge request с ревью. ([1:22:44](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4964))
- Манифесты генерируют общим Helm-чартом из шаблонов. ([1:20:30](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4830))
- В Git секрет лежит плейсхолдером, а значение для каждого окружения подставляет CI из переменных проекта. ([1:24:55](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=5095))
- Другой путь - ссылка на секрет в Vault: контроллер в кластере достаёт значение и подменяет манифест на лету. ([1:25:43](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=5143))

## Приложение само сообщает о здоровье

Чтобы Kubernetes знал, работает ли приложение, оно должно отвечать на служебные URL, например `/ready` и `/live`. Без этого не сделать canary и другие стратегии деплоя. ([1:16:24](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4584))

| Проба     | Когда работает                    | Итог                                        |
| --------- | --------------------------------- | ------------------------------------------- |
| startup   | только на старте, до первого 200  | на 200 OK приложение становится Ready       |
| readiness | после startup, постоянно          | без 200 OK реплика теряет трафик            |
| liveness  | постоянно                         | без 200 OK Kubernetes перезапускает под     |

([1:17:11](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4631))

> [!note]- Пропущено
> устройство kind и настройка его registry, minikube на виртуалках, клонирование демо-приложения, проблемы с передачей кода через чат Толка, OpenLens и подобные инструменты для наблюдения за кластером, синтаксис `kubectl scale`, `watch kubectl get pods`, поды `kube-system`, формат имени пода, ДЗ.
