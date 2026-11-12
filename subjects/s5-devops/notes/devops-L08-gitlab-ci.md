---
date: 2026-11-06
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9
---
# L08. GitLab CI: сборка, тесты, публикация и деплой

Практика: с нуля собран `.gitlab-ci.yml` для приложения с пробами и Helm-чартом из [[devops-L07-ci-cd-git-helm|L07]]. Код восстановлен по трансляции, имена образов и неймспейсов условные.

## Пайплайн описан кодом в репозитории

`.gitlab-ci.yml` - CI как код: обычный файл в репозитории, обычно рядом с кодом приложения. Отдельно его держат, когда один CI обслуживает много приложений или им управляет отдельный DevOps. ([08:26](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=506))

```yaml
stages: [build, test, publish, deploy]    # порядок стейджей = порядок запуска

default:
  tags: [local, docker]                   # применяется ко всем джобам, джоба может переопределить

build-job:
  stage: build                            # ровно один стейдж на джобу
  variables:
    PROBES_LOCAL_IMAGE: "probes-app:${CI_PROJECT_ID}-${CI_PIPELINE_ID}"
  script:                                 # каждая строка - отдельная команда
    - docker build -t "${PROBES_LOCAL_IMAGE}" ./probes-app
```

**Стейдж** - смысловая группа джоб (сборка, тесты, деплой); джобы одного стейджа стартуют параллельно. ([09:07](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=547))

**Джоба** - имя, стейдж и `script`, строки которого выполняет шелл раннера: bash на Linux, PowerShell на Windows. У Docker executor команды идут в контейнере из `image` джобы или из образа по умолчанию в настройках раннера. ([12:30](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=750))

- Раннеры сами раз в несколько секунд спрашивают у API GitLab джобы под свои теги (pull-модель). ([17:37](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=1057))
	- если у раннера выключен запуск джоб без тегов, джоба без `tags` вечно висит в очереди
- `CI_PROJECT_ID`, `CI_PIPELINE_ID` - встроенные переменные: тег с ними уникален для каждого пайплайна. ([15:38](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=938))

## Джобы обмениваются данными через артефакты

Джобы могут идти на разных раннерах, поэтому файлы и переменные между ними передают только артефактами. **Артефакт** - бинарный неверсионируемый файл, который GitLab хранит после джобы: его можно скачать или передать следующим джобам. ([26:52](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=1612))

```yaml
test-job:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:10.0
  dependencies: []                        # артефакты других джоб не нужны
  script:
    - dotnet test --logger "junit;LogFilePath=test-results/results.xml"
  artifacts:
    when: always                          # отчёт нужен и при упавших тестах
    reports:
      junit: test-results/results.xml     # GitLab рисует вкладку Tests

build-job:
  # ... как выше, плюс:
  script:
    - docker build -t "${PROBES_LOCAL_IMAGE}" ./probes-app
    - echo "PROBES_LOCAL_IMAGE=${PROBES_LOCAL_IMAGE}" > build.env
  artifacts:
    paths: [build.env]                    # только чтобы посмотреть файл при отладке
    reports:
      dotenv: build.env                   # переменные из файла получат следующие джобы
```

- По умолчанию артефакты джоб прошлых стейджей скачиваются во все следующие джобы. ([46:42](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=2802))
	- `dependencies` сужает список, `dependencies: []` отключает скачивание
- `needs` задаёт порядок между джобами и тоже передаёт их артефакты. ([1:12:15](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4335))
- Сырой XML в браузере GitLab не покажет, для этого есть `reports`. ([31:13](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=1873))
	- JUnit XML выдаёт тестовый фреймворк, в .NET - через пакет-логгер
- Через dotenv-отчёт по пайплайну передают, например, вычисленную semver-версию. ([46:42](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=2802))

## Временный тег для тестов, версия для реестра

Образ собирают с временным тегом пайплайна, чтобы ничего не сломать на проде, а в реестр он уходит перетегированным под версию релиза. Деплой берёт ту же версию: если она разъедется с публикацией, на окружение уедет не тот образ. ([43:10](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=2590), [1:09:21](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4161))

```yaml
variables:
  PROBES_APP_VERSION: "v8"                # глобальная: нужна и publish, и deploy

publish-image:
  stage: publish
  dependencies: [build-job]               # отсюда приходит PROBES_LOCAL_IMAGE
  script:
    - docker tag "${PROBES_LOCAL_IMAGE}" "<registry>/probes-app:${PROBES_APP_VERSION}"
    - docker push "<registry>/probes-app:${PROBES_APP_VERSION}"
```

- Переменные задают на уровне джобы, всего файла или проекта в настройках. Переменную одной джобы правильнее держать в ней. ([52:50](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3170))
- Релизную версию по идее присваивают при мёрже в master; в демо она меняется руками. [?] как это совмещается с правилом "в master только деплой", не показано. ([58:08](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3488))
- В Контуре скрипты джоб спрятаны в подключаемые YAML-шаблоны: разработчики задают только переменные. ([55:33](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3333))

## Деплой из CI: доступ в кластер и очередь

Джобе деплоя нужны Helm и доступ в кластер; доступ даёт kubeconfig из переменной проекта типа File. В настройках проекта хранят только то, что нельзя положить в репозиторий, а секреты - в Vault ([[devops-L05-deployment-ресурсы-конфиги#Настройки и секреты приходят снаружи образа|L05]]). ([59:06](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3546))

```yaml
deploy-staging:
  stage: deploy
  image:
    name: <образ с helm и kubectl>        # версии инструментов зафиксированы
    entrypoint: [""]                      # не запускать неизвестный entrypoint образа
  needs: [publish-image]
  resource_group: staging                 # деплои в одно окружение идут по очереди
  environment:
    name: staging
  variables:
    NAMESPACE: "probes-staging"
  script:                                 # KUBECONFIG = путь к файлу из переменной проекта
    - |
      helm upgrade --install probes-app ./chart -n "${NAMESPACE}" --create-namespace \
        --set image="<registry>/probes-app:${PROBES_APP_VERSION}" --wait --timeout 3m
      kubectl get deployments,pods,services -n "${NAMESPACE}"
```

**Переменная типа File** - в окружении джобы лежит не значение, а путь к временному файлу с ним. kubectl читает `KUBECONFIG` как путь, а многострочный файл с пробелами в обычной переменной окружения непредсказуемо ломается. ([1:02:24](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3744))

- **Защищённая переменная** раскрывается только в пайплайнах защищённых веток. ([1:04:08](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3848))
- Версии Helm и kubectl в образе фиксируют: свежий Helm при старом кластере может сломать деплой и откат. ([1:04:58](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3898))
- Два одновременных `helm upgrade` в одно окружение ломают релиз. `resource_group` ставит джобы с одной группой в очередь даже из разных пайплайнов, для деплоя это обязательно. ([1:13:32](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4412))
- Образ задают через `--set`: `values.yaml` лежит в коде и меняться на каждый пайплайн не может. ([1:08:47](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4127))

## Окружения и правила разводят стейджинг и прод

Деплой на стейджинг после каждого коммита - уже continuous delivery; прод идёт только после стейджинга, а между ними должен стоять quality gate ([[devops-L07-ci-cd-git-helm#CI, delivery и deployment - три ступени автоматизации|L07]]). Какая ветка что делает, задают `rules`: в демо из master только деплоят на прод, а в остальных ветках прода нет. ([1:16:05](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4565), [1:21:58](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4918))

```yaml
deploy-staging:
  rules:
    - if: $CI_COMMIT_BRANCH && $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH

deploy-production:                        # image и script как у deploy-staging
  stage: deploy
  resource_group: production
  environment:
    name: production
  variables:
    NAMESPACE: "probes-production"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual                        # кнопка для релиз-инженера
```

- `rules` решают, создавать ли джобу; build, test и publish получают такое же правило, как стейджинг. ([1:24:59](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5099))
- По лектору, общее поведение задают в `workflow`, а джобы его переопределяют. По документации `workflow:rules` решает, создавать ли пайплайн целиком; повтор правил в джобах убирают через `extends` или YAML-якоря (вне конспекта, [workflow:rules](https://docs.gitlab.com/ci/yaml/#workflowrules)). ([1:25:54](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5154))
- `needs` на джобу, которую `rules` не создали, валит пайплайн: в master прод не может зависеть от стейджинга. ([1:27:53](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5273))
- Вместо ручного запуска прод можно катить по расписанию, например ночью. ([1:29:23](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5363))
- В master попадают только через merge request, который вливается после зелёного пайплайна. ([1:26:25](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5185))

**Окружение** (`environment`) - цель деплоя в GitLab: видно, кто и когда туда деплоил, можно передеплоить или остановить. ([1:30:12](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5412))

- Переменные проекта ограничивают окружением: у прода и стейджинга разные kubeconfig. ([1:31:01](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5461))
- Остановка запускает описанную джобу со `action: stop`, например `helm uninstall`. ([1:31:53](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5513))
- Имя окружения и неймспейс можно строить из имени ветки: получается [[devops-L07-ci-cd-git-helm|динамический стенд]], который удаляют при остановке ради экономии ресурсов. ([1:17:29](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4649))

> [!note]- Пропущено
> требования стенда GitLab Omnibus к памяти, настройка VS Code/VSCodium с плагинами YAML и GitLab Workflow, каталог компонентов в редакторе пайплайнов, отладка опечаток в демо, Hadolint перед сборкой (задача со звёздочкой), хранение артефактов в Контуре в S3 с retention, дубли имён джоб в YAML, `untracked: true` в артефактах (не делать), опции `access` и `expire_in` (по умолчанию 30 дней с сервера), переезд с TeamCity и его автоподстановка логгеров, кавычки в YAML и `${VAR}` (в акцентах курса), отступление про `100` и `"100m"` CPU в Kubernetes, `needs` на build при наличии publish (лишний), маскирование file-переменных, bash-скрипты деплоя в духе Mail.ru, ДЗ (CI/CD своего приложения), передача курса команде телеметрии.
