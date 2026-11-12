---
date: 2026-09-25
ingest: fast
links:
  - https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?usp=sharing
---
# P02. Основы Kubernetes

Манифесты со слайдов и из архива семинара, сокращены.

## От серверов к оркестратору

Каждый этап инфраструктуры решал проблему предыдущего. ([[sre-P02-transcript|05:19]])

| Этап          | Как устроен                                    | Что мешало                              |
| ------------- | ---------------------------------------------- | --------------------------------------- |
| Физ. серверы  | один сервер - одно приложение                  | железо простаивает, нет изоляции        |
| Виртуализация | гипервизор запускает несколько ОС на сервере   | накладные расходы на каждую гостевую ОС |
| Контейнеры    | общее ядро хоста, изоляция на уровне процессов | множеством контейнеров трудно управлять |

**Контейнер** - обычный процесс на хосте, ограниченный cgroups и namespaces, поэтому он лёгкий и быстро стартует. ([[sre-P02-transcript|11:03]])

Когда контейнеров сотни на многих серверах, нужен оркестратор с пятью задачами: балансировка, восстановление упавших, обновление без даунтайма с откатом, сеть с обращением по имени, хранилище, переживающее перезапуск. ([[sre-P02-transcript|14:54]])

Kubernetes стал стандартом, потому что: ([[sre-P02-transcript|18:17]])
1. Зародился в Google на опыте Borg и Omega.
2. Расширяем (свои типы ресурсов, CRD; операторы; CNI, CSI) и оброс экосистемой.
3. Описывает систему декларативно.

## Желаемое состояние и control loop

**Декларативность** - описываем желаемое состояние, а Kubernetes сам приводит к нему систему и дальше следит за ним. ([[sre-P02-transcript|17:41]])

Состояние лежит только в etcd, остальные компоненты его не хранят, поэтому их можно перезапускать и масштабировать горизонтально. ([[sre-P02-transcript|23:22]])

| Компонент                | Где     | Роль                                                              |
| ------------------------ | ------- | ----------------------------------------------------------------- |
| etcd                     | control | распределённое консистентное key-value хранилище состояния        |
| API-server               | control | REST API, единственный ходит в etcd; аутентификация, авторизация  |
| Controller Manager       | control | контроллеры сверяют реальное состояние с манифестами              |
| Scheduler                | control | назначает поды на ноды по ресурсам, affinity, приоритету          |
| cloud-controller-manager | control | в облаке создаёт ноды, балансировщики, диски через API провайдера |
| kubelet                  | data    | на каждой ноде запускает поды через CRI и перезапускает упавшие   |
| kube-proxy               | data    | реализует Service правилами iptables или IPVS                     |

Компоненты следят (watch) за изменениями через API-server и через него же пишут результат. Запуск пода по слайду: ([[sre-P02-transcript|01:29:01]])

```text
user -> API: create ReplicaSet             -> etcd
Controller Manager видит новый ReplicaSet  -> API: создать поды -> etcd
Scheduler видит поды без nodeName          -> API: bind на ноду -> etcd
kubelet ноды видит свой под                -> docker run -> статус в API -> etcd
```

В kubectl `create` императивна, а `apply` декларативна. ([[sre-P02-transcript|01:32:12]])

```bash
kubectl create -f file.yaml           # только создаёт; если объект есть - ошибка
kubectl apply -f file.yaml            # создаст или обновит; повторять безопасно
kubectl get pod -o wide
kubectl describe pod my-pod
kubectl exec -it my-pod -- /bin/sh    # оболочка в контейнере; безопасники часто запрещают
kubectl logs my-pod -c sidecar        # логи из stdout; -c выбирает контейнер пода
kubectl set image deployment my-deployment nginx=nginx:1.13
kubectl delete -f file.yaml
```

## Под одноразовый, его держат контроллеры

**Pod** - минимальная единица развёртывания: тесно связанные контейнеры с общим IP и портами, по сути группа процессов на ноде. ([[sre-P02-transcript|36:07]])

**Sidecar** - контейнер рядом с основным, который дополняет его, не вмешиваясь: логер, прокси, агент мониторинга. ([[sre-P02-transcript|38:20]])

**Init-контейнер** - запускается до основных, последовательно и один раз; пока он не завершится успешно, основной не стартует.

Упавший под Kubernetes сам не пересоздаст, поэтому поды вручную не создают: их держат контроллеры. ([[sre-P02-transcript|40:59]])

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp           # свои поды контроллер находит по лейблам
  template:
    metadata:
      labels:
        app: myapp         # должны совпадать с селектором
    spec:
      containers:
      - name: nginx
        image: nginx:1.29.1
  strategy:
    type: RollingUpdate    # или Recreate
    rollingUpdate:
      maxSurge: 1          # не больше 1 пода сверх replicas
      maxUnavailable: 1    # не больше 1 недоступного
```

| Контроллер  | Что держит                                     | Для чего                            |
| ----------- | ---------------------------------------------- | ----------------------------------- |
| ReplicaSet  | ровно `replicas` подов с нужными лейблами      | самоисцеление                       |
| Deployment  | ReplicaSet'ы, через них поды                   | stateless-приложения                |
| StatefulSet | поды с постоянными номером, DNS-именем и томом | БД и другие приложения с состоянием |
| DaemonSet   | по поду на каждой ноде                         | агенты мониторинга и логов, CNI     |
| Job         | поды до успеха или лимита попыток              | миграции БД, разовые задачи         |
| CronJob     | Job по расписанию crontab                      | периодические задачи                |

- ReplicaSet при смене образа старые поды не трогает
- Deployment при смене образа создаёт новый ReplicaSet, старый остаётся с 0 подов
- StatefulSet создаёт поды по порядку номеров, удаляет в обратном
- сбор логов DaemonSet'ом: [[sre-P03-логирование#Сбор и доставка]]

## Конфигурация отдельно от образа

Один образ запускают во всех окружениях, а различия (адреса БД, креды, домены) лежат в ConfigMap, как требует фактор 3 из [[sre-P01-devops-sre-12-факторов#Конфигурация и окружение]]. ([[sre-P02-transcript|58:01]])

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
data:
  username: k8s-admin
  access_level: "1"
---
# в контейнере пода, два способа
envFrom:                     # 1. ключи как переменные окружения
- configMapRef:
    name: myconfigmap
volumeMounts:                # 2. ключи как файлы; том config ссылается на ConfigMap
- name: config
  mountPath: /etc/nginx/conf.d/
```

Файл из тома обновляется без перезапуска пода, но приложение должно само заметить изменение; nginx за файлом не следит, сигнал ему может подавать sidecar. Без ConfigMap под не запустится. ([[sre-P02-transcript|01:00:31]])

**Secret** - объект для паролей, токенов, доступа к docker-registry и TLS-сертификатов. ([[sre-P02-transcript|01:03:25]])

Secret не шифрует, а только кодирует в base64. Защищают его шифрованием etcd, ограничением доступа через RBAC или внешними хранилищами вроде Vault. ([[sre-P02-transcript|01:05:16]])

## Где живут данные

В Kubernetes обычно запускают stateless-сервисы, а СУБД и объектные хранилища держат вне его, по Мусину. ([[sre-P02-transcript|01:47:29]])

| Том          | Где данные                       | Сколько живут                |
| ------------ | -------------------------------- | ---------------------------- |
| hostPath     | каталог на ноде                  | дольше пода, но на этой ноде |
| emptyDir     | RAM или диск ноды, сначала пуст  | пока жив под; для кэша       |
| PV через PVC | диск или внешнее хранилище (NFS) | независимо от пода           |

- при переезде пода на другую ноду данные hostPath теряются

**PersistentVolume (PV)** - хранилище, доступное кластеру. ([[sre-P02-transcript|01:10:10]])

**PersistentVolumeClaim (PVC)** - заявка на хранилище нужного класса и объёма.

**StorageClass** - тип хранилища, зарегистрированный администратором.

Пользователь создаёт PVC, контроллеры привязывают к ней подходящий PV (статус Bound, только к одной PVC), и под ссылается на PVC. ([[sre-P02-transcript|01:11:44]])

## Сеть: Service и Gateway API

IP пода меняется при перезапуске, а реплик несколько. **Service** даёт группе подов стабильный IP и DNS-имя и балансирует трафик между ними. ([[sre-P02-transcript|01:14:19]])

**Namespace** - логическая группа ресурсов кластера, например окружение; без указания ресурс попадает в `default`. ([[sre-P02-transcript|56:47]])

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  selector:
    app: myapp        # поды Deployment'а
  ports:
  - protocol: TCP
    port: 80          # порт сервиса
    targetPort: 80    # порт в поде
# DNS: myservice.default.svc.cluster.local
```

| Тип                      | Где виден                                           |
| ------------------------ | --------------------------------------------------- |
| ClusterIP (по умолчанию) | только внутри кластера                              |
| NodePort                 | на порту ноды, например в minikube                  |
| LoadBalancer             | снаружи через балансировщик облака, он платный      |
| Headless                 | без своего IP: DNS отдаёт IP подов, для StatefulSet |

ExternalName лектор отнёс к доступу через внешний балансировщик; по [документации](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) это DNS-псевдоним (CNAME) на внешнее имя (вне конспекта).

**Gateway API** - набор CRD для входа внешнего трафика в кластер, сменил Ingress. Gateway встречает запросы снаружи и маршрутизирует, а Service балансирует уже внутри. ([[sre-P02-transcript|01:21:42]])

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway                 # точка входа, по сути балансировщик
metadata:
  name: prod-web
spec:
  gatewayClassName: envoy     # какой балансировщик за ним
  listeners:
  - protocol: HTTP
    port: 80
    name: prod-web-gw
    allowedRoutes:
      namespaces:
        from: Same            # маршруты только из своего namespace
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute               # куда направить трафик: по пути, заголовкам, методу
metadata:
  name: foo
spec:
  parentRefs:
  - name: prod-web
  rules:
  - backendRefs:
    - name: foo-svc           # Service, за ним поды
      port: 8080
```

> [!note]- Пропущено
> карьера семинариста, байка про сисадмина и ночные выкладки, снапшоты и миграция VM, год выпуска Kubernetes, kubelet вне контейнера, замена kube-proxy на eBPF, формат имён подов, детали Job (backoffLimit, activeDeadlineSeconds) и манифест DaemonSet node-exporter, пробы и ресурсы в манифестах архива, способы защиты Secret подробно, Ingress, TLS и аутентификация на Gateway, демо приложения oncall в minikube, вопросы про фронт и бэк в разных контейнерах, ДЗ.
