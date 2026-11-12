---
date: 2026-10-23
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel
---
# L06. Service, пробы, стратегии деплоя, Ingress

Практика на локальном кластере: связать клиентов с репликами, научить Kubernetes видеть здоровье приложения, обновлять его без простоя и опубликовать наружу. Манифесты восстановлены по трансляции, команды частично мои.

## Service даёт репликам одно имя и адрес

У каждого пода свой IP, и при перезапуске он меняется, а port-forward ведёт в один под. Service держит постоянный IP и имя во внутреннем DNS кластера, а поды находит по селектору. ([08:26](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=506), [10:18](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=618))

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web              # DNS-имя: клиенты ходят на http://web
spec:
  type: ClusterIP        # IP внутри кластера; другие типы не разбирали
  selector:
    app: web             # те же лейблы, что у подов Deployment
  ports:
    - name: http         # только имя: Service работает на L4
      port: 80           # порт, на который приходят клиенты
      targetPort: 8080   # порт контейнера
```

**EndpointSlice** - объект с IP и портами подов, куда Service пересылает трафик. При 4 репликах в нём 4 адреса, после scale до 2 - два. ([17:44](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=1064))

Service не балансирует нагрузку: под выбирается на каждое новое соединение, и все запросы внутри соединения идут в тот же под. Выбор идёт по round robin и равномерности не гарантирует, балансировка - задача L7-инструментов вроде nginx и Ingress-контроллера. ([19:30](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=1170), [24:25](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=1465))

```bash
kubectl run curl-client --image=curlimages/curl --restart=Never -- sleep 3600
# Connection: close на каждый запрос, иначе все 20 уйдут в один под
kubectl exec curl-client -- sh -c \
  'for i in $(seq 20); do curl -s -H "Connection: close" http://web/; sleep 1; done'
kubectl get endpointslices
```

## Три пробы отвечают на разные вопросы

Таблица проб - в [[devops-L05-deployment-ресурсы-конфиги#Приложение само сообщает о здоровье]]. Без startup-пробы Service шлёт запросы в только что запущенный под, и пока приложение прогревается, клиент получает 503. ([28:55](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=1735))

```yaml
# в контейнере Deployment
ports:
  - name: http
    containerPort: 8080
startupProbe:
  httpGet: { path: /live, port: http }   # port - имя из ports или номер
  periodSeconds: 2
  timeoutSeconds: 1
  failureThreshold: 30     # не поднялось за 30 попыток - считается нерабочим
livenessProbe:
  httpGet: { path: /live, port: http }
  periodSeconds: 2
  timeoutSeconds: 1
  failureThreshold: 3      # 3 ответа не 200 - kubelet перезапускает контейнеры
readinessProbe:
  httpGet: { path: /live, port: http }   # в демо тоже /live, чтобы уронить запросом
```

- Startup-проба работает только до перехода в Ready, liveness и readiness - после неё. ([34:35](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2075))
- Liveness перезапускает контейнеры только не ответившего пода; под остаётся тем же, растёт счётчик рестартов. ([37:47](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2267))
- По лектору, liveness-проб может быть несколько на разные пути приложения. По документации у контейнера одна проба каждого типа (вне конспекта, [Probe](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#lifecycle-1)).

Readiness-проба не перезапускает под, а выводит его из топологии: Service перестаёт слать в него трафик. Так перегруженная реплика успевает разобрать свою очередь, а вернувшись в Ready, снова получает запросы. ([39:14](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2354), [43:56](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2636))

## Стратегия деплоя опирается на пробы

Без проб система деплоя не знает, работает ли новая реплика, и обновлять по одной реплике бессмысленно. Kubernetes - платформа, а система деплоя на нём требует проб со стороны приложения. ([45:24](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2724))

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate    # или Recreate: удалить все поды и поднять заново
    rollingUpdate:
      maxSurge: 1          # сколько новых реплик сверх replicas
      maxUnavailable: 0    # сколько реплик может лежать; можно "50%"
```

- С такими настройками пользователю всегда доступны 4 реплики: старая гасится, только когда новая стала Ready. ([48:24](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2904))
- Разрешённое Kubernetes использует целиком: при `maxUnavailable: 50%` он сразу гасит 2 реплики из 4. ([53:40](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3220))
- По лектору, Recreate - стратегия по умолчанию. По документации по умолчанию RollingUpdate (вне конспекта, [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)).

**Blue-green** - новая версия поднимается целиком рядом со старой, и балансировщик переключает пользователей на неё. Лектор назвал Recreate похожим на blue-green [?]: по его же описанию Recreate сначала гасит все поды, а blue-green держит обе версии. ([46:20](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2780))

RollingUpdate - лишь подобие canary: настоящий canary вводит одну реплику, наблюдает за ней и только потом обновляет остальные. Kubernetes так не умеет, нужны Argo CD и подобные; в Контуре для этого писали свой контроллер, который спрашивал о реплике внутренний health check API. ([50:37](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3037), [51:57](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3117))

## Ingress публикует сервисы наружу на L7

**Ingress-контроллер** - приложение в кластере, которое принимает запросы извне и передаёт их сервисам. **Ingress** - объект-публикация с правилами для контроллера; контроллер один, публикаций много. ([1:04:12](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3852), [1:06:52](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4012))

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: traefik    # какой из контроллеров обслуживает
  rules:
    - host: web.localhost      # совпадение по заголовку Host
      http:
        paths:
          - path: /            # можно делить и по пути
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

- Правила смотрят на заголовки HTTP, это отличает L7 от L4. Запрос без подходящего Host получает 404. ([1:08:43](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4123), [1:17:06](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4626))
- В терминах nginx Service играет роль upstream - списка адресов, куда уходит трафик. ([1:10:52](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4252))
- Делить трафик по весам можно, если умеет контроллер: в nginx через аннотации, в Envoy и Gateway API - на ходу. Так пускают 1-5% на новую версию. ([1:11:49](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4309), [1:13:04](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4384))
- Сервис контроллера имеет тип NodePort, потому что в него ходят снаружи прямо на ноды. TCP вроде SSH тоже выставляют через NodePort. ([1:03:30](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3810), [1:05:23](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3923))
- Контроллер - административная часть, его ставят в отдельный неймспейс, а не рядом с приложениями. ([1:01:58](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3718))

nginx тяжёл для кластера: при новой конфигурации он либо рвёт все соединения, страдают например веб-сокеты, либо держит их со старыми подами. Поэтому берут лёгкие Traefik или Envoy Gateway; Контур переходит на Envoy. ([57:10](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3430))

## Helm - шаблонизатор и система деплоя

Чарт содержит шаблоны Deployment, Service и прочего, их пишет автор чарта, а пользователь передаёт только настройки в `values.yaml`. Helm ещё и ждёт, пока приложение станет Ready, поэтому связка проб и Helm даёт настоящий continuous deployment. ([1:24:26](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=5066), [1:26:23](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=5183))

```bash
helm repo add traefik https://traefik.github.io/charts   # где лежат чарты
helm repo update
helm upgrade --install traefik traefik/traefik \
  -n ingress-system --create-namespace -f values.yaml     # установка и обновление
```

## NetworkPolicy - файрвол внутри кластера

**NetworkPolicy** - разрешающие и запрещающие правила для входящего (ingress) и исходящего (egress) трафика подов по селектору. Работает на уровне IP, а не DNS, и в пределах неймспейса. ([1:30:46](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=5446))

## Хеш образа против подмены

Злоумышленник может пересобрать образ с другими правами или библиотекой и запушить под тем же тегом, но хеш SHA-256 изменится. Поэтому хорошая практика - проверять хеш образа, деплоя его на стенды. ([03:23](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=203))

> [!note]- Пропущено
> удаление стенда удалением неймспейса вместо `kubectl delete -f`, отладка повисшего port-forward, демо-приложение с эмуляцией смерти, `kubectl get events`, параметры чарта Traefik в values.yaml, поиск лишнего порта 8080 в демо, вопрос про helmfile, просьба о слайдах, ДЗ.
