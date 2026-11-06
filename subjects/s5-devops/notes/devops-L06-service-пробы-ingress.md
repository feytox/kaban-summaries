---
date: 2026-10-23
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel
---
# L06. Практика Kubernetes: Service, пробы, стратегии деплоя, Ingress

Практика в kind. Service и Gateway API подробнее в [[sre-P02-основы-kubernetes#Сеть|SRE P02]]. Команды и манифесты восстановлены по рассказу.

## По итогам ДЗ

- снести стенд целиком - `kubectl delete namespace <ns>`: `delete -f <dir>` и `delete all` оставляют ConfigMap, Secret и объекты из других файлов ([01:09](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=69))
- **digest образа** (sha256) защищает от атаки на цепочку поставок: злоумышленник перепушит образ под тем же тегом, но digest изменится; хорошая практика - сверять digest образа, который катят на все стенды ([03:23](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=203))

## Service

Service держит постоянный IP (тип ClusterIP по умолчанию) и регистрируется во внутреннем DNS под своим именем; поды находит по `selector`. Переименовал Service - клиенты, ходящие по старому имени, сломались. ([06:58](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=418))

```yaml
kind: Service
metadata: {name: web}
spec:
  selector: {app: web}
  ports:
    - name: http        # имя любое: Service работает на L4, протокол ему не важен
      port: 80          # порт Service
      targetPort: 8080  # порт приложения в поде
```

- **EndpointSlice** - объект с IP и портами подов, куда Service шлёт трафик: 4 реплики - 4 адреса (`kubectl get endpointslices`) ([17:44](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=1064))
- **Service не балансирует нагрузку**: он выбирает случайный под для нового соединения, и все запросы одного соединения (keep-alive, port-forward) идут в тот же под ([19:30](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=1170))
- равномерную балансировку дают L7-инструменты: nginx, HAProxy, ingress-контроллер

```bash
kubectl run curl-client --image=curlimages/curl --restart=Never -- sleep 3600
# Connection: close - новое соединение на каждый запрос, иначе всё время один под
kubectl exec curl-client -- sh -c \
  'for i in $(seq 20); do curl -s -H "Connection: close" http://web/; sleep 1; done'
```

## Пробы

Для проб приложение должно отвечать на служебные пути: тестовое отдаёт `/ready`, `/live`, первые ~10 с после старта отвечает 503 и умеет "притвориться мёртвым". Это минимум со стороны приложения для любых стратегий деплоя. ([27:26](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=1646))

| Проба     | Когда работает           | Не прошла                                             |
| --------- | ------------------------ | ----------------------------------------------------- |
| startup   | только до первого успеха | под не получает трафик, пока приложение прогревается  |
| liveness  | после startup, постоянно | kubelet перезапускает контейнеры этого пода           |
| readiness | после startup, постоянно | под убирают из EndpointSlice Service, без перезапуска |

([34:05](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2045))
- на демо liveness перезапустила контейнеры, а под остался тем же, вырос счётчик рестартов ([37:47](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2267))
- readiness нужна, например, когда реплика перегружена и должна разгрести очередь: на время её выводят из балансировки ([39:14](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2354))
- проб может быть несколько, на разные методы API

```yaml
containers:
  - name: web
    ports: [{name: http, containerPort: 8080}]
    startupProbe:
      httpGet: {path: /live, port: http}  # port - имя из ports или номер
      periodSeconds: 2
      timeoutSeconds: 1
      failureThreshold: 30                # 30 попыток x 2 с на старт
    livenessProbe:
      httpGet: {path: /live, port: http}
      periodSeconds: 2
      failureThreshold: 3
    readinessProbe:
      httpGet: {path: /ready, port: http}
```

## Стратегии деплоя

Без проб стратегия деплоя не важна: система деплоя не знает, работает ли новая реплика и можно ли обновлять следующую. ([45:05](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2705))

- **Recreate** удаляет все поды и поднимает новые
	- по лектору, это стратегия по умолчанию; по документации по умолчанию RollingUpdate ([docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)) (вне конспекта)
- **Blue-green** - новая версия целиком поднимается рядом со старой, после проверки балансировщик переключает пользователей
	- по лектору, Recreate похож на blue-green; Recreate гасит старые поды до запуска новых и даёт простой, а blue-green нет (вне конспекта)
- **RollingUpdate** с `maxSurge: 1, maxUnavailable: 0`: при 4 репликах поднимается пятая, после `Ready` гасится старая, и так по одной; пользователю всегда доступно 4 ([47:19](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=2839))
	- значения можно задать в процентах; это не границы, а указание: при `maxUnavailable: 50%` сразу гасится половина реплик
	- новая реплика не работает - пользователь не страдает, старые на месте

RollingUpdate - неполный canary: настоящий canary аккуратно вводит одну реплику, наблюдает за ней и только потом обновляет остальные. "Понаблюдать за новой репликой" Kubernetes из коробки не умеет; это делают Argo CD и подобные системы деплоя. В Контуре написали свой CRD-контроллер: он спрашивал о реплике Health check API и только через какое-то время отдавал пробе 200. ([51:57](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3117))

## Ingress

([55:54](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3354))
- **ingress-контроллер** принимает запросы снаружи кластера и ведёт их через Service в поды, на L7
- контроллер ставят в отдельный namespace (`ingress-system`), не рядом с приложениями; его Service - NodePort, потому что в него ходят снаружи
- в проде контроллеры стоят на отдельных серверах с белыми IP, и DNS-имя сайта указывает на них ([01:15:52](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4552))
- **nginx как ingress** тяжёлый: при новом конфиге рвёт все соединения (страдает WebSocket), а без перезапуска держит соединения со старыми подами ([57:10](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=3430))
- Контур переходит на Envoy Gateway ([[sre-P02-основы-kubernetes#Сеть|Gateway API]]); на практике - лёгкий Traefik
- **Ingress** (публикация) - правило маршрутизации; контроллер один или несколько разных, публикаций много, нужный контроллер выбирают через `ingressClassName` ([01:06:52](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4012))
- доли трафика (0.1% на новую реплику, 30/70) умеет Envoy, у других контроллеров - аннотации, если поддерживают ([01:11:49](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=4309))
- TCP, например SSH, наружу выставляют через NodePort

```yaml
kind: Ingress
spec:
  ingressClassName: traefik
  rules:
    - host: web.localhost        # сверяется заголовок Host
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: {service: {name: web, port: {number: 80}}}
```

```bash
helm repo add traefik https://traefik.github.io/charts && helm repo update
helm upgrade -i traefik traefik/traefik -n ingress-system --create-namespace -f values.yaml
curl -H 'Host: web.localhost' http://localhost:8080/   # Host не совпал ни с одним rule -> 404
```

## Helm

**Helm** - шаблонизатор манифестов и инструмент деплоя: автор чарта уже описал Deployment, Service и прочее, ты передаёшь только `values.yaml`. ([01:23:26](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=5006))
- Helm ждёт, пока приложение станет `Ready`, и при неудаче откатывает на прошлые конфиги и секреты
- **связка проб и Helm** даёт настоящий continuous deployment ([01:26:23](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=5183))

## NetworkPolicy

**NetworkPolicy** - фаервол для подов: правила ingress и egress для подов по `podSelector`, действует в пределах namespace и работает по IP, а не по DNS. ([01:29:34](https://fiit.ktalk.ru/recordings/PvwOjMp2SEsSTyNFGVel?t=5374))

> [!note]- Пропущено
> отладка зависшего port-forward, поиск, откуда взялся порт 8080, helmfile, ДЗ.
