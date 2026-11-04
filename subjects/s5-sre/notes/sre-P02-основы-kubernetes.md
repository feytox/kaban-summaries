---
date:
ingest: full
links:
  - https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view
---
# P02. Основы Kubernetes

## Путь к Kubernetes

Каждый этап решал проблему предыдущего: ([05:19](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=319))

| Этап               | Суть                                 | Минус                                    |
| ------------------ | ------------------------------------ | ---------------------------------------- |
| Физические серверы | один сервер = одно приложение        | железо простаивает, нет изоляции         |
| Виртуальные машины | гипервизор запускает несколько ОС    | каждая гостевая ОС ест ресурсы           |
| Контейнеры         | общее ядро хоста, изоляция процессов | сотнями контейнеров не управлять вручную |

**Контейнер** - обычный процесс на хосте, ограниченный cgroups и namespaces, а не виртуальная машина. Поэтому контейнер с другим ядром (Windows) на Linux-хосте не запустить. ([11:03](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=663))

Оркестратор нужен, когда контейнеров сотни на многих серверах: он балансирует трафик, поднимает упавшее, обновляет без даунтайма, даёт общую сеть и хранилище. ([14:54](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=894))

Почему победил Kubernetes: ([18:17](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1097))
- вырос из внутренних систем Google (Borg, Omega) и вышел уже проверенным на огромной нагрузке
- расширяем на каждом уровне (CRD, операторы, CNI, CSI), вокруг большая экосистема
- декларативность: описываешь желаемое состояние, Kubernetes приводит к нему систему и следит, чтобы оно сохранялось

## Архитектура кластера

**Control plane** управляет кластером, **data plane** - ноды с нагрузкой. Роли лучше держать на разных нодах. ([21:40](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1300))

| Компонент                | Где     | Что делает                                                             |
| ------------------------ | ------- | ---------------------------------------------------------------------- |
| etcd                     | control | key-value хранилище, единственное место состояния кластера             |
| API server               | control | REST API, единственный вход в кластер и единственный, кто пишет в etcd |
| Controller manager       | control | контроллеры приводят кластер к манифестам                              |
| Scheduler                | control | выбирает ноду для нового пода                                          |
| cloud-controller-manager | control | только в облаке: создаёт ноды, балансировщики, диски                   |
| kubelet                  | data    | агент на каждой ноде, запускает контейнеры через CRI                   |
| kube-proxy               | data    | сетевые правила (iptables, IPVS) на каждой ноде, реализует Service     |

- остальные компоненты stateless, поэтому их можно перезапускать и масштабировать ([23:25](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1405))
- контроллер - **control loop**: сверяет реальное состояние с желаемым постоянно, а не только при `apply` ([26:09](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1569))
- Scheduler сам ничего не запускает: отсеивает ноды без нужных ресурсов, из оставшихся берёт лучшую, учитывает affinity и приоритет, пишет `nodeName` в под ([27:31](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1651))
- kubelet - единственный компонент, который работает не в контейнере, а системным процессом ([29:55](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1795))
- kube-proxy можно заменить CNI-плагином вроде Cilium или Calico ([32:00](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1920))

Запуск пода: kubectl -> API server пишет объект в etcd -> контроллер создаёт под -> Scheduler назначает ноду -> kubelet этой ноды запускает контейнеры и сообщает статус. Все шаги идут через API server. ([01:29:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5341))

## Pod

**Pod** - минимальная единица развёртывания: один или несколько тесно связанных контейнеров с общим IP и портами, друг к другу они ходят через `localhost`. ([36:03](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2163))

**Sidecar** - контейнер рядом с основным, который дополняет его, не вмешиваясь в работу: сборщик логов, прокси, агент мониторинга. ([38:20](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2300))

**Init-контейнер** - готовит окружение до старта основных и завершается. Init-контейнеры идут по очереди, по разу; если какой-то упал, основной контейнер не стартует. ([38:55](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2335))

Под одноразовый: если упала нода, сам под не пересоздастся. Это делают контроллеры, поэтому поды вручную не создают. ([40:59](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2459))

## Контроллеры нагрузки

**ReplicaSet** - держит заданное число подов и поднимает новый вместо упавшего или удалённого. ([41:31](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2491))
- поды выбирает по label selector, поэтому labels шаблона должны совпадать с селектором
- новый образ получат только новые поды: старые продолжат работать со старым ([44:24](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2664))

**Deployment** - контроллер над ReplicaSet, который добавляет обновления без даунтайма и откат; основная абстракция для приложений. Обновление = поменять тег образа и применить манифест. ([45:25](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2725))
- стратегии `Recreate` и `RollingUpdate`
- `maxSurge: 1`, `maxUnavailable: 1` - меняем поды по одному

**StatefulSet** - группа подов для приложений с состоянием: у каждого пода постоянный номер (`mysql-0`), DNS-имя и свой том. Поды создаются по порядку номеров и удаляются в обратном. ([48:11](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2891))

**DaemonSet** - по поду на каждой ноде, в том числе на новых. ([50:25](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3025))

| Контроллер  | Для чего                | Пример                            |
| ----------- | ----------------------- | --------------------------------- |
| Deployment  | stateless-приложения    | веб-приложение                    |
| StatefulSet | приложения с состоянием | БД                                |
| DaemonSet   | служебные агенты        | node-exporter, сборщик логов, CNI |

**Job** - запускает поды разовой задачи, пока она не завершится успешно или не кончатся попытки `backoffLimit`. Лимит не даёт сломанной задаче перезапускаться бесконечно. Примеры: миграции БД, отчёты. ([53:49](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3229))

**CronJob** - создаёт Job по расписанию в синтаксисе crontab. ([55:40](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3340))

## Namespace, ConfigMap, Secret

**Namespace** - логическое разделение ресурсов кластера, например по окружениям; без указания объект попадает в `default`. ([56:47](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3407))

**ConfigMap** - конфигурация в виде ключ-значение отдельно от кода, чтобы один образ работал во всех окружениях. ([58:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3481))
- файлом через volume: обновляется без перезапуска пода, но приложение должно само перечитать файл (можно поручить sidecar'у)
- переменными окружения через `envFrom.configMapRef`
- если ConfigMap нет, под не запустится

**Secret** - для паролей и токенов приложения, доступа к закрытому registry, TLS-сертификатов. ([01:03:25](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3805))

Secret не шифрует, а кодирует в Base64: кто может прочитать объект, видит значение. Защита: шифрование в etcd (только хранилище), доступ через RBAC, внешние Sealed Secrets или Vault. ([01:05:16](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3916))

## Хранение данных

| Том          | Где данные                     | Живёт              |
| ------------ | ------------------------------ | ------------------ |
| emptyDir     | на ноде, изначально пусто      | пока жив под       |
| hostPath     | директория на ноде             | пока жива нода     |
| PV через PVC | хранилище кластера (NFS, диск) | независимо от пода |

([01:07:14](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4034))
- emptyDir - для кэша и промежуточных файлов
- с hostPath данные теряются, когда под переезжает на другую ноду

**PersistentVolume (PV)** - конкретное хранилище кластера. **PersistentVolumeClaim (PVC)** - заявка пода на хранилище нужного типа (**StorageClass**) и объёма. Когда PVC привязан к PV, тот получает статус `Bound`, а под монтирует PVC. ([01:10:10](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4210))

## Сеть

**Service** - стабильный IP и DNS-имя `<service>.<namespace>.svc.cluster.local` для группы подов с балансировкой между ними. Нужен, потому что IP пода меняется при перезапуске. Поды выбирает по label selector; `port` - порт Service, `targetPort` - порт в поде. ([01:14:19](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4459))

| Тип            | Доступен                   | Когда                        |
| -------------- | -------------------------- | ---------------------------- |
| `ClusterIP`    | только внутри кластера     | по умолчанию, связь сервисов |
| `NodePort`     | снаружи через порт на ноде | доступ в minikube            |
| `LoadBalancer` | через балансировщик облака | managed-кластер в облаке     |

([01:15:41](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4541))
- по семинаристу, NodePort открывает порт на ноде, где запущена нагрузка, и пробрасывает его в под
- по документации, NodePort - надстройка над ClusterIP: создаётся cluster IP, и один и тот же порт (по умолчанию 30000-32767) открывается на каждой ноде ([документация](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)) (вне конспекта)
- `LoadBalancer` в облаке платный: одна строка в манифесте раздувает счёт
- `ExternalName` на слайде стоит рядом с `LoadBalancer`; по документации он ничего не балансирует, а отдаёт CNAME на внешнее имя (вне конспекта)

**Headless Service** - Service без своего IP: DNS отдаёт адреса всех подов, у каждого пода своё имя. Нужен для обращения к конкретному поду, обычно со StatefulSet. ([01:20:08](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4808))

**Gateway API** - CRD для входа внешнего трафика в кластер, замена устаревшему Ingress. Service балансирует внутри кластера, Gateway принимает трафик снаружи (и может терминировать TLS); он дополняет Service, а не заменяет. ([01:21:42](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4902))
- цепочка: `Gateway` (точка входа: класс балансировщика, порт) -> `HTTPRoute` (правила по пути, заголовкам) -> Service -> поды
- `allowedRoutes.namespaces.from: Same` не пускает к Gateway маршруты других команд из чужих namespace

## kubectl

**kubectl create** императивно создаёт объект и падает, если он уже есть. **kubectl apply** декларативно приводит кластер к файлу, поэтому его можно запускать сколько угодно раз. ([01:32:12](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5532))

```bash
kubectl apply -f k8s/
kubectl get all -n <ns>; kubectl describe pod <pod>
kubectl exec -it <pod> -- sh
kubectl logs <pod> -c <container>    # -c, если в поде есть sidecar
kubectl scale deployment <name> --replicas 3
kubectl port-forward svc/<name> 8888:8080
```

- `exec` в проде часто запрещает безопасность: это неконтролируемый доступ внутрь сервиса ([01:35:07](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5707))
- `logs` видит только то, что приложение пишет в stdout

В демо приложение собирают и загружают в minikube через `minikube image load <образ>`, иначе образа там нет. Свой MySQL в Deployment семинарист сам назвал ошибкой: правильнее StatefulSet. ([01:39:06](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5946))

По куратору, в Kubernetes обычно запускают stateless-сервисы, а то, что хранит данные (СУБД, объектное хранилище), живёт вне его. Основные команды kubectl нужно знать обязательно. ([01:47:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=6421))

Пропущено: снапшоты и миграция VM, подробности etcd и cloud-controller-manager, именование подов Deployment, манифесты со слайдов, архив примеров манифестов, демо OnCall по шагам, вопрос про разделение фронта и бэка, ДЗ.
