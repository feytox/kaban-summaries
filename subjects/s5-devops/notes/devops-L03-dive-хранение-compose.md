---
date: 2026-10-02
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz
---
# L03. Docker: dive, хранение данных, Compose

Dockerfile на работе правят редко: в готовом проекте он уже написан, и меняют его только при смене языка, способа деплоя или архитектуры. Поэтому важнее понимать, как устроены образ и контейнер, чем помнить команды. ([05:49](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=349))

## dive: слои образа наглядно

**dive** - утилита, которая распаковывает образ и показывает по каждому слою команду, размер и снимок файловой системы. С ней видно, что принёс базовый образ, куда легли файлы и как каждая инструкция меняет файловую систему ([[devops-L02-docker-образы-и-слои#Слои и кэш сборки]]). ([08:53](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=533))

**Эффективность образа** - оценка dive, сколько места потрачено зря: файл, который трогают в нескольких слоях (добавили в одном, удалили в другом), занимает место в каждом. В демо "плохой" Dockerfile набрал 77%, а "хороший", где установка и очистка идут в одном `RUN`, почти 100% при меньшем размере. ([12:03](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=723))

Смотреть образ в dive после каждой правки бессмысленно, но проверку можно встроить в CI: пороги задаёт команда, как покрытие тестов, и неоптимальный Dockerfile ловится автоматически. ([15:02](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=902))

```bash
dive myapp:bad                           # интерактивно: слои слева, файлы слоя справа
CI=true dive myapp:good --ci-config .dive-ci   # exit code 0, если пороги пройдены
```

```yaml
# .dive-ci
rules:
  lowestEfficiency: 0.95
  highestUserWastedPercent: 0.10
```

- По порогам лектора не прошёл даже стандартный `python:3.14-slim`. ([17:36](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1056))

## Размер образа против времени сборки

Варианты базового образа (`slim` и другие) различаются набором заранее установленных библиотек и бинарников. Если приложению нужны библиотеки на C (numpy, ML, распознавание изображений), с `slim` итоговый образ выйдет почти того же размера, а сборка заметно удлинится: всё выпиленное придётся доустанавливать. ([22:18](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1338))

Уменьшая размер, увеличиваешь время сборки, и наоборот; лектор советует упор на время. Сборка идёт на каждый коммит, и пока сотня разработчиков ждёт пайплайн, компания платит за их простой больше, чем за диски раннеров. ([39:05](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2345))

- Выбирать под приложение: собрать оба варианта, замерить холодную сборку без кэша и размер в dive. ([25:14](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1514))
- На приватных раннерах компании базовые образы обычно уже лежат, и сборка берёт их из кэша. ([38:22](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2302))
- Multi-stage для Python (в итоге только интерпретатор и код) сжал образ с 400 до 136 МБ, но лектор считает это странной оптимизацией. ([20:04](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1204))

## Один образ на все окружения

Лектор рекомендует собирать образ один раз и прогонять его через все окружения: тесты, тестовые стенды, ручную проверку, прод. Тогда пользователь получает ровно тот код и окружение, что проверили автотесты и тестировщик. ([32:03](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1923))

Противоположный подход - отдельный образ под каждый этап: с pytest для тестов, с mypy и ruff для линтеров, без каталога тестов для прода. Он не тащит в прод лишнее, но проверенный образ уже не тот, что выкатили. ([33:39](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2019))

- Инструменты разработки (линтеры) в прод-образ не ставят и при первом подходе. ([32:03](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1923))
- В Точке первая стадия multi-stage задаёт окружение сборки: адрес registry, переменные пакетного менеджера, путь к сертификатам компании; её слои не попадают в итоговый образ. ([29:01](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1741))

## Volume и bind mount

Контейнер эфемерен, поэтому данные, которые должны его пережить, выносят в хранилище вне контейнера ([[devops-L02-docker-образы-и-слои#Данные контейнера]]). Volume и bind mount различаются тем, кто управляет данными. ([41:25](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2485))

```bash
docker volume create --label project=demo app-data
docker run --rm -v app-data:/data alpine:3.24 sh -c 'echo hello > /data/message.txt'
docker run --rm -v app-data:/data alpine:3.24 cat /data/message.txt   # данные пережили контейнер

docker run --rm -v "$(pwd)/src:/app" myapp:1.0   # bind mount: правки кода видны без пересборки
```

**Volume** - отдельный объект, которым управляет Docker: данные лежат в его внутреннем формате, их достают, подключив volume к контейнеру. Один volume можно подключить к нескольким контейнерам, но общая файловая система грозит гонками, так что это не best practice. ([44:11](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2651))

**Bind mount** - директория хоста, которую контейнер видит как свою. Нужен для локальной разработки: приложение с hot reload перезапускается при правке кода без пересборки образа, в проде почти не применяется. ([46:43](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2803))

- Частая ошибка с bind mount: без `USER` контейнер создаёт файлы от root, и на хосте их не прочитать без `sudo`. ([49:42](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2982))
- **tmpfs** - хранилище в оперативной памяти для секретов, полученных из Vault и подобных систем; умирает вместе с контейнером. ([54:24](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3264))
- Флаг `-v` у обоих одинаковый: имя слева от `:` означает volume, путь хоста - bind mount ([volumes](https://docs.docker.com/engine/storage/volumes/), [bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)) (вне конспекта). На паре вопрос остался без ответа.

## Docker Compose

**Docker Compose** - оркестрация на минималках: в одном файле описывают целевое состояние всех контейнеров приложения вместо длинных `docker run` с кучей флагов. Его используют, чтобы одной командой поднять проект с базой и nginx локально, а в проде оркестрирует Kubernetes. ([56:39](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3399))

```yaml
services:
  app:
    build: .
    environment:
      REDIS_HOST: cache            # имя сервиса и есть DNS-имя
    depends_on:
      db: {condition: service_healthy}     # стартовать после готовности базы
      cache: {condition: service_healthy}
    restart: unless-stopped        # перезапуск упавшего контейнера
    deploy:
      resources:
        limits: {cpus: "0.5", memory: 256M}   # те же cgroups
    networks: [back, front]
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD", "pg_isready"]  # готов, когда команда вернула 0
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s            # время на прогрев до первых проверок
    volumes: [db-data:/var/lib/postgresql/data]
    networks: [back]               # порт на хост не пробрасываем
  cache:
    image: redis:8
    healthcheck: {test: ["CMD", "redis-cli", "ping"]}
    networks: [back]
  web:
    image: nginx:1.29
    ports: ["80:80"]
    depends_on: [app]
    networks: [front]              # из nginx база недоступна
networks: {back: {}, front: {}}
volumes: {db-data: {}}
```

```bash
docker compose up -d --build   # в фоне, пересобрав изменённые образы
docker compose logs -f app     # логи сервиса в реальном времени
docker compose exec app sh     # зайти в контейнер
```

- Compose поднимает изолированную сеть, где контейнеры находят друг друга по DNS-именам сервисов: IP каждый раз новые. ([1:01:28](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3688))
- Volume подключают только к Postgres: Redis здесь просто кэш, его данные не нужны. ([1:06:17](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3977))

Код восстановлен по рассказу, конфиг `.dive-ci` с порогами и образы в compose мои.

> [!note]- Пропущено
> отзывы о ДЗ, dive не учитывает кэш сборки, проброс порта базы для приложения на хосте, интерфейс dive по шагам, `debug-netshoot` в compose лектора, можно ли Compose раскидать по нескольким VM (лектор предположил, что нет), жив ли Docker Swarm (лектор не помнит), ДЗ.
