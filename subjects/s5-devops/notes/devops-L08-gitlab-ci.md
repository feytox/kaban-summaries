---
date: 2026-11-06
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9
---
# L08. Практика GitLab CI: сборка, тесты, публикация, деплой

Пайплайн для приложения с пробами из [[devops-L06-service-пробы-ingress|L06]] и чарта из [[devops-L07-ci-cd-git-helm#Helm|L07]] на локальном GitLab и kind. `.gitlab-ci.yml` восстановлен по рассказу и собран в один файл.

## Основы

([08:26](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=506))
- **CI как код**: `.gitlab-ci.yml` лежит в репозитории приложения; отдельно его держат, когда CI один на много приложений или им владеет отдельный DevOps
- правят в ветке, не в main; писать удобнее в VS Code или VSCodium с плагинами YAML и GitLab Workflow
- **stage** группирует jobs по смыслу (build: бэкенд, фронтенд, образ); job относится ровно к одному stage ([09:07](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=547))
- две jobs с одним именем: YAML с одинаковыми ключами, выполнится только одна; раньше это не подсвечивалось
- `script` построчно уходит в shell раннера (bash на Ubuntu, PowerShell на Windows); у Docker executor - в контейнер из образа по умолчанию, который задан в раннере ([12:30](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=750))
- **pull-модель**: раннер сам периодически спрашивает API GitLab, есть ли jobs под его теги ([17:37](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=1057))
	- пайплайн завис в очереди - у jobs нет тегов, а у раннера выключено "Run untagged jobs"
- переменные пишут как `${VAR}`: в `$CI_PROJECT_ID-$CI_PIPELINE_ID` минус может попасть в имя переменной ([16:40](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=1000))
- **кавычки**: строки на ревью просят брать в кавычки; число без них YAML поймёт как число ([33:09](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=1989))
	- в Kubernetes `cpu: 100` - это 100 ядер, а `100m` - 0.1 ядра

## Пайплайн

```yaml
stages: [build, test, publish, deploy]

variables:
  PROBES_APP_VERSION: "v8"            # глобальная: publish и deploy обязаны брать одну версию

default:
  tags: [local-docker]                # применяется ко всем jobs, если они не переопределят

build_job:
  stage: build
  variables:
    PROBES_LOCAL_IMAGE: "probes-app:${CI_PROJECT_ID}-${CI_PIPELINE_ID}"  # временный тег
  script:
    - docker build -t "${PROBES_LOCAL_IMAGE}" ./probes-app
    - echo "PROBES_LOCAL_IMAGE=${PROBES_LOCAL_IMAGE}" > build.env
  artifacts:
    reports:
      dotenv: build.env               # переменные из файла получат следующие jobs

test_job:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:10.0  # своё окружение вместо образа раннера
  dependencies: []                    # артефакты не скачивать
  script:
    - dotnet test --logger "junit;LogFilePath=test-results.xml"
  artifacts:
    when: always                      # отчёт нужен и при упавших тестах
    reports:
      junit: test-results.xml         # вкладка Tests в пайплайне

publish_image:
  stage: publish
  needs: [build_job]                  # порядок и артефакты build_job
  script:
    - docker tag "${PROBES_LOCAL_IMAGE}" "${REGISTRY}/probes-app:${PROBES_APP_VERSION}"
    - docker push "${REGISTRY}/probes-app:${PROBES_APP_VERSION}"

deploy_staging:
  stage: deploy
  image:
    name: <образ с helm и kubectl>    # версии инструментов зафиксированы
    entrypoint: [""]                  # не запускать неизвестный entrypoint образа
  needs: [publish_image]
  environment: {name: staging}
  resource_group: staging             # второй деплой в это окружение ждёт в очереди
  variables: {NAMESPACE: staging}
  script:
    - |
      helm upgrade --install probes-app ./charts/probes-app \
        -n "${NAMESPACE}" --create-namespace \
        --set-string image="${REGISTRY}/probes-app:${PROBES_APP_VERSION}" \
        --wait --timeout 3m
      kubectl -n "${NAMESPACE}" get deployments,pods,services
  rules:
    - if: $CI_COMMIT_BRANCH && $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH

deploy_production:
  # image и script как у deploy_staging
  environment: {name: production}
  resource_group: production
  variables: {NAMESPACE: production}
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual                    # кнопку жмёт релиз-инженер
```

`REGISTRY` - адрес локального registry (имя моё). Такие же `rules`, как у `deploy_staging`, нужны build, test и publish.

## Артефакты и передача данных

([26:52](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=1612))
- **artifacts** - файлы из job: скачать себе или передать в другие jobs; в Контуре GitLab хранит их в S3 с retention
- по умолчанию предлагает `untracked: true` (всё, чего нет в индексе Git) - так не делать, указывать `paths`
- `expire_in` по умолчанию берётся с сервера (30 дней)
- **reports** - артефакты, которые GitLab понимает: `junit` рисует тесты, `dotenv` передаёт переменные ([31:13](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=1873))
- JUnit-отчёт должен уметь выдавать тестовый фреймворк: для .NET нужен пакет JunitXml.TestLogger, Go умеет из коробки
- артефакты по умолчанию уходят во все последующие jobs ([46:42](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=2802))
- через dotenv по пайплайну носят, например, SemVer-версию с хешем коммита в patch
- `dependencies` - чьи артефакты скачать, `[]` - никаких; `needs` - явный порядок между jobs, граф виден в Visualize ([01:12:15](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4335))
	- по лектору, `dependencies` тоже задаёт порядок; сам он потом поправился, что порядок задаёт `needs`
	- `needs` на build не нужен, если он уже есть у publish

## Публикация и версии

([43:10](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=2590))
- локальный образ живёт с временным тегом и никуда не пушится, чтобы не сломать прод
- в registry уходит перетегированный образ с релизной версией
- такое тегирование по-хорошему делают только при merge в master
- **переменные** задают на уровне job, всего CI или проекта (Settings -> CI/CD); переменную одной job держат в этой job
- в Контуре script-секции спрятаны в шаблоны: разработчик подключает их и задаёт только переменные ([55:33](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3333))
- длинный скрипт выносят в `deploy.sh`; так делали в Mail.ru на Jenkins ([01:10:05](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4205))

## Деплой

([01:00:43](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3643))
- в настройках проекта хранят только то, что нельзя положить в репозиторий; секреты - в Vault, в проекте лишь доступ к нему
- **переменная типа File**: в окружении job путь к временному файлу с содержимым, а не само значение
	- переменная `KUBECONFIG` типа File - и kubectl с Helm сами найдут кластер; маскирование для неё бесполезно
	- kubeconfig указывает на DNS-имя мастер-ноды: у раннера и кластера свои localhost
- **protected-переменная** раскрывается только в пайплайнах защищённых веток ([01:04:08](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3848))
- образ для деплоя собирают сами с фиксированными версиями Helm и kubectl: свежий Helm может не работать со старым кластером ([01:04:58](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=3898))
- по лектору, `|` - конструкция bash; это литеральный блок YAML (вне конспекта)
- **resource_group** обязателен для деплоев: два одновременных Helm-деплоя в одно окружение ломают релиз ([01:13:11](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4391))

## Окружения и правила

([01:17:29](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4649))
- **environment** разделяет окружения; для динамического стенда имя и namespace берут из имени ветки
- остановку окружения описывает job с `action: stop`: её скрипт удаляет релиз и namespace, освобождая ресурсы
- в Operate -> Environments видно, кто и когда деплоил; там же передеплой и остановка ([01:30:12](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5412))
- переменной можно задать scope окружения: у staging и production разные kubeconfig
- деплой на прод идёт после staging (`needs`), между ними должен стоять quality gate ([01:20:58](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4858))
- **rules** решают, создавать ли job; решение лектора: из master только деплой на прод, в остальных ветках всё, кроме него ([01:21:58](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=4918))
	- `workflow` задаёт поведение всего пайплайна, чтобы не копировать правила в каждую job
	- в master `needs` на несозданную job ломает пайплайн, и dotenv оттуда не придёт: прод деплоит конкретную версию ([01:27:53](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5273))
- в master попадают через MR с условием успешного пайплайна
- `when: manual` - ручной запуск, ещё можно деплоить по расписанию ночью ([01:28:36](https://fiit.ktalk.ru/recordings/QGX2enzTiUy4LDgBKOE9?t=5316))

Пропущено: проверка стендов на 8 ГБ, отладка опечаток в YAML, ДЗ.
