---
date: 2026-09-25
ingest: full
links:
  - https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view
---
# P02. Основы Kubernetes

Манифесты сокращены, часть примеров составлена мной.

## Зачем Kubernetes

**Контейнер** - обычный процесс на хосте, изолированный cgroups и namespaces; ядро общее с хостом, это не VM. ([11:03](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=663))

Когда контейнеров сотни на многих серверах, вручную ими не управлять. Оркестратор берёт на себя балансировку, перезапуск упавшего, обновление без простоя, сеть и хранилище. ([14:54](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=894))

Почему Kubernetes стал стандартом: ([18:17](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1097))
- вырос из внутренних систем Google (Borg, Omega), то есть вышел уже проверенным на огромной нагрузке
- расширяется на любом уровне: CRD, операторы, CNI, CSI
- **декларативный**: описываешь желаемое состояние, он приводит к нему кластер и следит, чтобы оно сохранялось

## Архитектура

([21:40](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1300))

| Компонент          | Где           | Роль                                                |
| ------------------ | ------------- | --------------------------------------------------- |
| etcd               | control plane | key-value хранилище всего состояния кластера        |
| API server         | control plane | единственный вход в кластер, только он пишет в etcd |
| Controller manager | control plane | контроллеры приводят объекты к манифестам           |
| Scheduler          | control plane | выбирает ноду для нового пода                       |
| kubelet            | каждая нода   | запускает контейнеры подов через CRI                |
| kube-proxy         | каждая нода   | сетевые правила (iptables, IPVS) для Service        |

- все компоненты, кроме etcd, stateless, поэтому их можно перезапускать и масштабировать
- контроллер - **control loop**: сверяет реальное состояние с желаемым постоянно, а не только при `apply` ([26:09](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1569))
- Scheduler сам ничего не запускает: отсеивает ноды без ресурсов, учитывает affinity и пишет в под `nodeName` ([27:31](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1651))
- kubelet - единственный компонент, который работает не в контейнере, а обычным системным процессом
- kube-proxy можно заменить CNI-плагином (Cilium, Calico)
- в облачных кластерах есть ещё cloud-controller-manager: создаёт ноды, балансировщики и диски через API облака

Запуск пода: ([01:29:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5341))

```
kubectl -> API server -> etcd
  -> контроллер создаёт под
  -> Scheduler записывает nodeName
  -> kubelet этой ноды запускает контейнеры, статус обратно через API server
```

## Pod

**Pod** - минимальная единица развёртывания: один или несколько тесно связанных контейнеров с общим IP и портами. ([36:03](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2163))

```yaml
kind: Pod
spec:
  initContainers:          # по очереди и до основных; упал -> основные не стартуют
    - name: migrate
      image: app:1.2
      command: ["app", "migrate"]
  containers:              # друг к другу ходят через localhost
    - name: app
      image: app:1.2
    - name: log-agent      # sidecar: дополняет основной, не вмешиваясь в его работу
      image: vector
```

- **sidecar** - вспомогательный контейнер рядом с основным: сборщик логов, прокси, агент мониторинга ([38:20](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2300))
- **init-контейнер** - готовит окружение и завершается: директории, проверка БД
- под одноразовый: если упала нода, он не пересоздастся, поэтому поды создают через контроллеры, а не вручную ([40:59](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2459))

## Контроллеры

**Deployment** - держит N подов через ReplicaSet, обновляет без простоя и откатывает; основная абстракция для приложений. ([45:25](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2725))

```yaml
kind: Deployment
spec:
  replicas: 3
  selector:
    matchLabels: {app: web}   # свои поды ищет по labels
  template:
    metadata:
      labels: {app: web}      # обязаны совпасть с selector
    spec:
      containers:
        - name: web
          image: web:1.2      # новый тег + apply = обновление
  strategy:
    type: RollingUpdate       # Recreate - убить все и поднять заново
    rollingUpdate: {maxSurge: 1, maxUnavailable: 1}   # меняет поды по одному
```

- голый ReplicaSet новый образ к живым подам не применяет: обновятся только пересозданные ([44:24](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2664))
- после обновления старый ReplicaSet остаётся с 0 реплик, на него и откатываются

| Контроллер  | Для чего                      | Особенность                                                     |
| ----------- | ----------------------------- | --------------------------------------------------------------- |
| Deployment  | stateless-приложения          | поды взаимозаменяемы, имена случайные                           |
| StatefulSet | приложения с состоянием, БД   | номер пода (`mysql-0`), свои DNS-имя и том, создание по порядку |
| DaemonSet   | агенты: логи, мониторинг, CNI | по поду на каждой ноде, включая новые                           |

([51:46](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3106))

**Job** - запускает поды разовой задачи (миграции, отчёты), пока она не завершится успешно или не кончатся попытки. ([53:49](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3229))

```yaml
kind: Job
spec:
  backoffLimit: 4              # попыток до Failed; без лимита сломанная задача крутилась бы вечно
  activeDeadlineSeconds: 60    # работает дольше -> DeadlineExceeded
  template:
    spec:
      restartPolicy: Never
      containers: [...]
---
kind: CronJob
spec:
  schedule: "0 * * * *"        # crontab: каждый час
  jobTemplate: {...}           # внутри обычный Job
```

## Конфигурация

**Namespace** - логическая группа ресурсов в кластере, например под окружение; по умолчанию `default`. ([56:47](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3407))

**ConfigMap** - конфигурация ключ-значение отдельно от образа, чтобы один образ работал во всех окружениях. **Secret** - то же для паролей, токенов registry и TLS-сертификатов. ([58:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3481))

```yaml
kind: ConfigMap
metadata: {name: app-config}
data:
  LOG_LEVEL: info
---
# контейнер пода
envFrom:
  - configMapRef: {name: app-config}    # все ключи -> env; нет такого ConfigMap -> под не стартует
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef: {name: db, key: password}
volumeMounts:
  - {name: config, mountPath: /etc/nginx/conf.d/}   # ConfigMap файлами
volumes:
  - name: config
    configMap: {name: nginx-config}
```

- файл из ConfigMap обновляется без перезапуска пода, но приложение должно само его перечитать; это может делать sidecar
- Secret только кодируется в Base64, не шифруется: `kubectl get secret db -o yaml` покажет значение ([01:05:16](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3916))
- защита Secret: шифрование etcd (только на диске), RBAC, внешние Vault или Sealed Secrets

## Хранение

| Том          | Где данные                     | Живут                                     |
| ------------ | ------------------------------ | ----------------------------------------- |
| emptyDir     | на ноде, сначала пусто         | пока жив под; для кэша и временных файлов |
| hostPath     | в директории ноды              | на ноде; под на другой ноде их не видит   |
| PV через PVC | хранилище кластера (диск, NFS) | независимо от пода                        |

([01:07:14](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4034))

**PersistentVolume (PV)** - конкретное хранилище кластера. **PersistentVolumeClaim (PVC)** - заявка на хранилище нужного StorageClass и объёма; привязанный PV получает статус `Bound`, и под монтирует PVC как том. ([01:10:10](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4210))

## Сеть

**Service** - постоянный IP и DNS-имя для группы подов с балансировкой между ними. Нужен, потому что IP пода меняется при каждом перезапуске. ([01:14:19](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4459))

```yaml
kind: Service
metadata: {name: web}
spec:
  type: NodePort         # по умолчанию ClusterIP
  selector: {app: web}   # трафик идёт на поды с этим label
  ports:
    - port: 80           # порт Service
      targetPort: 8080   # порт в поде
# DNS: web.<namespace>.svc.cluster.local
```

| Тип          | Доступен                     | Когда                               |
| ------------ | ---------------------------- | ----------------------------------- |
| ClusterIP    | только внутри кластера       | связь сервисов между собой          |
| NodePort     | + снаружи через порт на ноде | доступ в minikube                   |
| LoadBalancer | + через балансировщик облака | облако; платный, легко раздуть счёт |

([01:15:41](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4541))

NodePort по семинаристу открывает порт на ноде, где запущена нагрузка. По документации он построен поверх ClusterIP и открывает один и тот же порт (по умолчанию 30000-32767) на каждой ноде ([документация](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)) (вне конспекта).

**Headless Service** - Service без своего IP (`clusterIP: None`): DNS отдаёт адреса всех подов, у каждого пода своё имя. Нужен для обращения к конкретному поду, обычно со StatefulSet. ([01:20:08](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4808))

**Gateway API** - CRD для входа внешнего трафика в кластер, замена устаревшему Ingress. Service балансирует внутри кластера, Gateway принимает трафик снаружи: они дополняют друг друга. ([01:21:42](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4902))

```yaml
kind: Gateway                 # точка входа
metadata: {name: prod-web}
spec:
  gatewayClassName: envoy     # какой балансировщик
  listeners:
    - {protocol: HTTP, port: 80, name: http,
       allowedRoutes: {namespaces: {from: Same}}}   # маршруты только из своего namespace
---
kind: HTTPRoute               # куда вести трафик
spec:
  parentRefs: [{name: prod-web}]
  rules:
    - backendRefs: [{name: foo-svc, port: 8080}]    # дальше Service -> поды
```

## kubectl

| Команда          | Подход        | Если объект уже есть           |
| ---------------- | ------------- | ------------------------------ |
| `kubectl create` | императивный  | ошибка                         |
| `kubectl apply`  | декларативный | обновит или скажет "unchanged" |

([01:32:12](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5532))

```bash
kubectl apply -f k8s/
kubectl get all -n <ns>
kubectl describe pod <pod>
kubectl exec -it <pod> -- sh          # в проде часто запрещён: неконтролируемый доступ
kubectl logs <pod> -c <container>     # только stdout; -c, если в поде есть sidecar
kubectl scale deployment <name> --replicas 3
kubectl set image deployment <name> web=web:1.3
kubectl port-forward svc/<name> 8888:8080
minikube image load <образ>           # иначе minikube не увидит локально собранный образ
```

По куратору, в Kubernetes обычно запускают stateless-сервисы, а СУБД и объектные хранилища чаще живут вне его. Основные команды kubectl нужно знать обязательно. ([01:47:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=6421))

Пропущено: этапы эволюции инфраструктуры, детали etcd и cloud-controller-manager, ExternalName, демо OnCall, вопрос про разделение фронта и бэка, ДЗ.
