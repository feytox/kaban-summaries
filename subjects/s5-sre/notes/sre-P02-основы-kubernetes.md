---
date: 2026-09-25
ingest: full
links:
  - https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view
---
# P02. Основы Kubernetes

## Зачем нужен Kubernetes

Инфраструктура прошла путь от физических серверов через виртуальные машины к контейнерам. Контейнеры лёгкие, но когда их сотни на многих серверах, вручную ими не управлять, и нужен оркестратор. ([05:19](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=319))

**Контейнер** - обычный процесс на хосте, изолированный через cgroups и namespaces. Это не виртуальная машина: ядро у всех контейнеров общее с хостом. ([11:03](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=663))

Оркестратор балансирует трафик, перезапускает упавшее, обновляет без простоя, связывает контейнеры сетью и даёт им хранилище. Kubernetes стал стандартом, потому что вырос из внутренних систем Google, легко расширяется и работает декларативно: вы описываете желаемое состояние, а он приводит к нему кластер и поддерживает его. ([18:17](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1097))

## Архитектура кластера

Компоненты делятся на **control plane**, который управляет кластером, и **data plane** - ноды, где работают приложения. ([21:40](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1300))

Control plane:
- **etcd** - key-value хранилище, единственное место, где лежит состояние кластера. Остальные компоненты состояния не хранят, поэтому их можно перезапускать и масштабировать.
- **API server** - единственный вход в кластер. Через него ходят kubectl и все компоненты, и только он пишет в etcd. ([24:13](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1453))
- **Controller manager** запускает контроллеры. Каждый контроллер постоянно сверяет реальное состояние своих объектов с манифестом и исправляет расхождения, а не только в момент `apply`. ([26:09](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1569))
- **Scheduler** выбирает ноду для нового пода, но сам его не запускает. Он отбрасывает ноды, где не хватает ресурсов, из остальных берёт лучшую с учётом affinity и приоритета и записывает её в под. ([27:31](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1651))
- **cloud-controller-manager** есть только в облачных кластерах. Через API облака он создаёт ноды, балансировщики и диски.

Data plane, на каждой ноде:
- **kubelet** запускает контейнеры назначенных ноде подов и следит за ними. Это единственный компонент, который работает не в контейнере, а обычным системным процессом. ([29:55](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=1795))
- **kube-proxy** настраивает сетевые правила, через которые работает Service. Его можно заменить CNI-плагином вроде Cilium.

Запуск пода проходит так: kubectl отправляет манифест в API server, контроллер создаёт под, Scheduler назначает ему ноду, kubelet этой ноды запускает контейнеры. Все шаги идут через API server и etcd. ([01:29:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5341))

## Pod

**Pod** - минимальная единица развёртывания: один или несколько тесно связанных контейнеров. У них общий IP и порты, поэтому друг к другу они обращаются через `localhost`. ([36:03](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2163))

Кроме основного контейнера, в поде бывают служебные:
- **Sidecar** работает рядом с основным контейнером и дополняет его, не вмешиваясь в работу: собирает логи, проксирует трафик, отдаёт метрики. ([38:20](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2300))
- **Init-контейнер** выполняется до основных и завершается: готовит директории, проверяет БД. Init-контейнеры идут по очереди, и если один упал, основной контейнер не запустится.

Под одноразовый: если его нода упала, сам он не пересоздастся. Пересоздают поды контроллеры, поэтому вручную поды не создают. ([40:59](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2459))

## Контроллеры нагрузки

Для постоянно работающих приложений:
- **ReplicaSet** держит заданное число подов и поднимает новый вместо упавшего. Свои поды он находит по labels, поэтому labels в шаблоне должны совпадать с селектором. Новый образ получат только новые поды, а старые так и будут работать со старым. ([41:31](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2491))
- **Deployment** управляет ReplicaSet'ами и добавляет обновление без простоя и откат. Чтобы обновить приложение, меняют тег образа и применяют манифест. Стратегия `RollingUpdate` заменяет поды постепенно, `Recreate` - все сразу. Для приложений почти всегда берут его. ([45:25](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2725))
- **StatefulSet** нужен приложениям с состоянием, например БД. У каждого пода постоянный номер (`mysql-0`), своё DNS-имя и свой том. Поды создаются по порядку номеров, а удаляются в обратном. ([48:11](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=2891))
- **DaemonSet** запускает по поду на каждой ноде, включая новые. Им запускают служебных агентов: node-exporter, сборщики логов, сетевые плагины. ([50:25](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3025))

Для разовых задач:
- **Job** запускает поды, пока задача не завершится успешно или не кончатся попытки (`backoffLimit`). Лимит не даёт сломанной задаче перезапускаться бесконечно. Так запускают миграции БД и отчёты. ([53:49](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3229))
- **CronJob** создаёт Job по расписанию в синтаксисе crontab.

## Конфигурация

**Namespace** - логическая группа ресурсов внутри кластера, например под одно окружение. Объект без namespace попадает в `default`. ([56:47](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3407))

**ConfigMap** хранит конфигурацию в виде пар ключ-значение отдельно от кода, чтобы один образ работал во всех окружениях. Передать её в под можно двумя способами: ([58:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3481))
- **Файлом через volume.** Файл обновляется без перезапуска пода, но приложение должно само заметить изменение и перечитать его.
- **Переменными окружения** через `envFrom.configMapRef`. Если такого ConfigMap нет, под не запустится.

**Secret** хранит пароли и токены приложения, доступ к закрытому registry и TLS-сертификаты. Данные в нём не шифруются, а только кодируются в Base64: кто может прочитать объект, видит значение. Защищают секреты шифрованием в etcd, ограничением доступа через RBAC или внешними инструментами вроде Vault. ([01:03:25](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=3805))

## Хранение данных

Тома различаются тем, сколько живут данные: ([01:07:14](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4034))

| Том          | Где данные                       | Сколько живут                                  |
| ------------ | -------------------------------- | ---------------------------------------------- |
| emptyDir     | на ноде, сначала пусто           | пока жив под                                   |
| hostPath     | в директории на ноде             | пока жива нода; под на другой ноде их не видит |
| PV через PVC | в хранилище кластера (диск, NFS) | независимо от пода                             |

**PersistentVolume (PV)** - конкретное хранилище в кластере. **PersistentVolumeClaim (PVC)** - заявка пода на хранилище нужного класса и объёма. Когда заявку привязывают к PV, под может смонтировать её как том. ([01:10:10](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4210))

## Сеть

**Service** даёт группе подов постоянный IP и DNS-имя `<service>.<namespace>.svc.cluster.local` и балансирует трафик между ними. Он нужен, потому что IP пода меняется при каждом перезапуске. Поды Service находит по labels, а `targetPort` указывает порт в поде. ([01:14:19](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4459))

Тип Service определяет, откуда он доступен: ([01:15:41](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4541))
- **ClusterIP** - только внутри кластера. Это тип по умолчанию, через него сервисы ходят друг к другу.
- **NodePort** - ещё и снаружи, через порт на ноде. Так обычно открывают доступ к приложению в minikube.
	- По семинаристу, порт открывается на ноде, где запущена нагрузка.
	- По документации, NodePort построен поверх ClusterIP и открывает один и тот же порт (по умолчанию из 30000-32767) на каждой ноде ([документация](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)) (вне конспекта).
- **LoadBalancer** - через внешний балансировщик облака. Он платный, и одна строка в манифесте может заметно увеличить счёт.

**Headless Service** - Service без собственного IP: DNS возвращает адреса всех подов, и у каждого пода есть своё имя. Его используют, когда нужно обратиться к конкретному поду, обычно вместе со StatefulSet. ([01:20:08](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4808))

**Gateway API** - набор ресурсов для входа внешнего трафика в кластер, замена устаревшему Ingress. Service балансирует трафик внутри кластера, а Gateway принимает его снаружи, поэтому они дополняют друг друга. Трафик идёт по цепочке `Gateway` -> `HTTPRoute` -> Service -> поды, где `HTTPRoute` задаёт правила маршрутизации по пути и заголовкам. ([01:21:42](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=4902))

## kubectl

`kubectl create` создаёт объект и падает с ошибкой, если он уже есть. `kubectl apply` приводит кластер к состоянию из файла, поэтому его можно запускать сколько угодно раз. ([01:32:12](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5532))

```bash
kubectl apply -f k8s/
kubectl get all -n <ns>
kubectl describe pod <pod>
kubectl exec -it <pod> -- sh
kubectl logs <pod> -c <container>    # -c нужен, если в поде есть sidecar
kubectl scale deployment <name> --replicas 3
kubectl port-forward svc/<name> 8888:8080
```

`kubectl logs` показывает только то, что приложение пишет в stdout. `exec` удобен для отладки, но в проде его часто запрещают, потому что он даёт неконтролируемый доступ внутрь сервиса. ([01:35:07](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=5707))

Локально собранный образ нужно загрузить в minikube командой `minikube image load <образ>`, иначе кластер его не найдёт. ([01:41:28](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=6088))

По куратору, в Kubernetes обычно запускают stateless-сервисы, а то, что хранит данные (СУБД, объектное хранилище), чаще живёт вне его. Основные команды kubectl после курса нужно знать обязательно. ([01:47:01](https://drive.google.com/file/d/1xjumD3HXVFcw_qsdi4eGqglLI7u1QWWy/view?t=6421))

Пропущено: этапы эволюции инфраструктуры подробно, детали etcd и cloud-controller-manager, ExternalName, параметры RollingUpdate, манифесты со слайдов, архив примеров манифестов, демо OnCall, вопрос про разделение фронта и бэка, ДЗ.
