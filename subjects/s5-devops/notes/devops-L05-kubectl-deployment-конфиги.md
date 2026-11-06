---
date: 2026-10-16
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk
---
# L05. Практика Kubernetes: kubectl, Deployment, ресурсы, конфиги

Практика в локальном кластере kind. Объекты и манифесты подробнее в [[sre-P02-основы-kubernetes|SRE P02]], здесь только то, чего там нет. Команды восстановлены по рассказу.

## Кластер и kubectl

**kind** - кластер Kubernetes в Docker-контейнерах: ноды - контейнеры, а не виртуалки, как у minikube. Registry поднимают рядом в отдельном контейнере, и ноды ищут его по имени хоста в общей сети, а не по localhost. ([04:38](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=278))

- **namespace** разделяет команды, чтобы одноимённые приложения не перетирали друг друга; системные компоненты живут в `kube-system`
- **контекст** - пользователь, кластер и namespace по умолчанию; хранится в `~/.kube/config`, сессий у kubectl нет ([12:28](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=748))

```bash
kubectl config use-context kind-hostinglab
kubectl config set-context --current --namespace=hostinglab  # больше не писать -n
kubectl get pods -A                # во всех namespace
watch kubectl get pods             # или -w: следить за изменениями
kubectl describe pod web           # IP, образ, Events: Scheduled, Pulled, Created, Started
kubectl get pods --show-labels
kubectl get pods -l app=web        # выборка по label
kubectl exec web -- env            # проверить, что переменные доехали
kubectl port-forward deployment/web 8081:8080   # через API server в один под, пока не Ctrl+C
```

## Requests и limits

([28:56](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=1736))

```yaml
resources:
  requests: {cpu: 100m, memory: 64Mi, ephemeral-storage: 64Mi}   # 1 ядро = 1000m
  limits:   {cpu: "1", memory: 256Mi, ephemeral-storage: 256Mi}
```

|             | requests                                  | limits                              |
| ----------- | ----------------------------------------- | ----------------------------------- |
| смысл       | минимум, без которого приложение не живёт | потолок                             |
| кто смотрит | scheduler выбирает по ним ноду            | ядро при работе                     |
| превышение  | -                                         | CPU: throttling; память: OOM killer |

- **requests указывать всегда**: иначе действуют дефолты кластера или namespace, о которых можно не знать ([31:42](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=1902))
- requests больше limits - манифест отклоняется
- OOM убивает главный процесс контейнера, а с ним все дочерние
- по лектору, за превышение ephemeral-storage тоже приходит OOM killer; по документации kubelet выселяет (evict) под ([docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#local-ephemeral-storage)) (вне конспекта)

## Pod, ReplicaSet, Deployment

([35:10](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2110))
- спеку живого пода менять нельзя: apply с новыми ресурсами падает, под надо удалить и создать заново; Deployment делает это сам, пересоздавая поды ([58:36](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3516))
- **ReplicaSet** собирает поды по `selector` (label `app: web`) и держит их число: удалишь под - тут же создаст новый
- цепочка: Deployment -> ReplicaSet -> Pod -> контейнер; к именам подов добавляется хэш
- в манифестах пишут только Deployment; голые Pod и ReplicaSet лектор показал в учебных целях ([01:28:44](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=5324))
- `kubectl scale` меняет кластер только до следующего apply из CI, поэтому все изменения идут через Git и CI ([47:56](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=2876))
- `kubectl rollout restart deployment web` перезапускает все реплики; с пробами - по одной
- почти любое изменение Deployment перезапускает поды, при одной реплике это простой

## ConfigMap и Secret

([52:35](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3155))
- `envFrom` берёт все ключи ConfigMap, `env` с `secretKeyRef` - отдельные ключи: так можно отдать API-ключ, не отдавая пароль из того же Secret
- один ConfigMap можно подключить ко всем приложениям namespace
- **зашивать настройки в образ** - по лектору, "путь террориста" ([01:00:15](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3615))

Secret лишь кодируется в Base64. От ConfigMap он отличается тем, что не выводится в консоль открытым текстом (`describe` показывает размер в байтах), права на него выдают отдельно (разработчик видит ConfigMap, но не Secret) и в нём удобно передавать файлы: многострочный приватный SSH-ключ, секретный конфиг. ([01:01:47](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=3707))

По лектору, изменённый ConfigMap не доходит до запущенных подов, и даже `rollout restart` не помог, понадобился передеплой. По документации переменные из ConfigMap читаются при старте контейнера, и новые значения подхватит любой перезапуск пода ([docs](https://kubernetes.io/docs/concepts/configuration/configmap/#configmaps-consumed-as-environment-variables)) (вне конспекта). Чтобы приложение перечитывало конфиг, ставят watcher-плагины, которые передеплоят Deployment при изменении ConfigMap; с Secret то же самое. ([01:10:37](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4237))

## Манифесты и секреты в Git

([01:20:30](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=4830))
- манифесты хранят в Git, рядом с кодом или в отдельном репозитории с CI
- изменение идёт через ветку, merge request и ревью ("зачем столько реплик?"); после merge пайплайн применяет конфиг
- руками манифесты не пишут: общий Helm-чарт генерирует их из шаблонов
- секреты в манифесте - плейсхолдеры, значения подставляет CI из переменных проекта, свои для каждого окружения ([01:24:06](https://fiit.ktalk.ru/recordings/AoXblXmzjU1TsuSuxDHk?t=5046))
- другой способ: в манифесте ссылка на секрет в Vault, контроллер в кластере достаёт его при деплое и подменяет манифест на лету; токен для Vault лежит в CI
- секреты в образе не держат: безопасники против

> [!note]- Пропущено
> настройка kind и registry из ДЗ, сборка и push образа, ручная правка Pod и ReplicaSet, обзор проб (в [[devops-L06-service-пробы-ingress|L06]]), ДЗ.
