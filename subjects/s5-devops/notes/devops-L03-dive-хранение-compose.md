---
date: 2026-10-02
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz
---
# L03. dive, хранение данных, Docker Compose

На работе Dockerfile обычно уже написан и правится редко, при смене языка или способа деплоя. Важнее понимать, как устроены образ и контейнер: это понадобится в Kubernetes. ([04:20](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=260))

## dive: слои изнутри

**dive** - утилита, которая раскладывает собранный образ по [[devops-L02-docker-образы-и-слои#Слои и кэш сборки|слоям]] и показывает, какие файлы добавил каждый слой. Она нужна, чтобы увидеть, куда что копируется, и найти лишнее. ([08:53](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=533))

**Эффективность образа** в dive - доля места, потраченного не впустую. Файл, тронутый в нескольких слоях (добавлен в одном, удалён в другом), считается потерей: у "плохого" Dockerfile с отдельными `RUN` вышло 77%, у "хорошего" с установкой и чисткой в одном слое - почти 100%. ([12:59](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=779))

```bash
dive myapp:1.0          # интерактивно: слои слева, файловая система справа
CI=true dive myapp:1.0  # без интерфейса: проверка порогов, exit code != 0 при провале
```

- В CI-режиме dive проверяет пороги, заданные в конфиге, и роняет пайплайн, если Dockerfile стал неоптимальным. ([15:56](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=956))
- Пороги каждый выбирает сам, как покрытие тестами: стандартный `python:3.14` не прошёл правила лектора. ([17:36](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1056))
- Эффективность считается только по файлам и размеру, кэш сборки dive не учитывает. ([18:26](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1106))

## Размер образа против времени сборки

Уменьшая образ, обычно удлиняют сборку, и наоборот. Лектор советует делать упор на время сборки: сборка идёт на каждый пуш, и ожидание 100 разработчиков стоит дороже дисков на раннерах. ([24:32](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1472))

- Варианты базового образа (`slim` и полный) различаются набором заранее установленных библиотек и бинарников. ([22:18](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1338))
- `slim` выигрывает для простого веб-сервера, но с numpy, ML и другими библиотеками на C образ почти не уменьшится, а сборка сильно замедлится: выкинутое придётся ставить самим. ([22:53](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1373))
- Выбор решает замер: два Dockerfile, холодная сборка без кэша, размер в dive. В Точке используют полные образы. ([25:14](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1514))
- На раннерах компании заранее лежат общие базовые образы, чтобы джобы не качали их с нуля. ([38:22](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2302))
- Multi-stage для Python, где в итоговый образ попадают только интерпретатор и код, ужал образ с 400+ до 136 МБ; лектор назвал такую оптимизацию странной. ([20:57](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1257))
- Отдельной стадией удобно готовить окружение (сертификаты, адреса registry, переменные для пакетного менеджера) и не тащить её слои в продакшен-образ. ([29:01](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1741))

## Один образ на все окружения

Лектор рекомендует собирать образ один раз и прогонять его через тесты, тестовые стенды и прод: тогда пользователь получит ровно тот код и то окружение, которые проверили. ([32:03](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1923))

Противоположный подход - отдельный образ на тесты, линтеры и прод, чтобы в проде не было тестов и инструментов разработки. Он тоже рабочий, но гарантии "прод = проверенное" не даёт. ([33:39](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2019))

Это то же требование, что фактор "сборка, релиз, запуск" в [[sre-P01-devops-sre-12-факторов#Код и сборка|12 факторах]].

## Хранение данных

[[devops-L02-docker-образы-и-слои#Данные контейнера|Три способа хранения]] различаются тем, кто управляет данными. ([51:21](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3081))

```bash
docker volume create st-data
docker run --rm -v st-data:/data alpine:3.24 sh -c 'echo "данные в томе" > /data/message.txt'
docker run --rm -v st-data:/data alpine:3.24 cat /data/message.txt   # данные пережили контейнер
docker run --rm -v "$PWD/work:/work" alpine:3.24 ls /work            # bind mount директории хоста
```

- **Volume** - отдельный объект Docker: внутрь с хоста не лезут, данные достают через подключённый контейнер. ([46:43](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2803))
- Один volume можно подключить к нескольким контейнерам, но общий доступ к файлам грозит гонками, поэтому так делать не советуют. ([45:01](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2701))
- **Bind mount** делит директорию хоста с контейнером: приложение с hot reload подхватывает правки кода без пересборки, в проде так почти не делают. ([47:33](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2853))
- Файлы, созданные контейнером через bind mount, принадлежат пользователю контейнера, по умолчанию root: на хосте их не прочитать без `sudo`. ([49:08](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2948))
- **tmpfs** живёт в оперативной памяти: туда кладут секреты из Vault и подобных систем, и они исчезают вместе с контейнером. ([54:24](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3264))
- Как `-v` отличает volume от bind mount, лектор не ответил: имя без `/` означает именованный volume, абсолютный путь - bind mount ([Docker docs](https://docs.docker.com/engine/storage/bind-mounts/)) (вне конспекта).

## Docker Compose

**Docker Compose** - оркестрация на минималках: один YAML-файл описывает целевое состояние всех контейнеров проекта вместо длинных команд `docker run`. Его используют для локального запуска приложения с БД и прокси, в проде оркестрирует Kubernetes. ([58:33](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3513))

```yaml
services:
  app:
    build: .                        # собрать из локального Dockerfile
    environment:
      REDIS_HOST: cache             # имя сервиса = DNS-имя внутри сети
    depends_on:
      db: {condition: service_healthy}     # стартовать после готовности БД
      cache: {condition: service_healthy}
    networks: [back, front]
    restart: unless-stopped
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD", "pg_isready"]   # exit 0 = сервис готов
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s             # время на прогрев до первых проверок
    volumes: [db-data:/var/lib/postgresql/data]   # данные пишет только БД
    networks: [back]
  cache:
    image: redis:8
    networks: [back]
  web:
    image: nginx:1.29
    ports: ["8080:80"]              # наружу торчит только nginx
    networks: [front]
networks: {back: {}, front: {}}
volumes: {db-data: {}}
```

- `depends_on` с `service_healthy` поднимает приложение только после того, как healthcheck БД и Redis вернул 0: без БД приложение бесполезно. ([1:00:09](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3609))
- Compose создаёт изолированную сеть, где сервисы находят друг друга по имени через DNS: IP каждый раз новые. ([1:01:28](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3688))
- Сети разделяют доступ: nginx в `front` видит приложение, но не БД. ([1:05:27](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3927))
- В Compose задают лимиты CPU и памяти (те же cgroups) и перезапуск упавшего контейнера. ([1:03:12](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3792))

```bash
docker compose up -d --build   # в фоне, пересобрать изменившееся
docker compose logs -f app     # логи сервиса в реальном времени
docker compose exec app sh     # команда внутри контейнера
```

Код восстановлен по демо, версии образов мои.

> [!note]- Пропущено
> опрос о сложности пары, обзор хороших практик Dockerfile из L02, интерфейс dive, Dockerfile в Точке (сертификаты), тесты в продакшен-образе Точки, labels у volume, Compose на нескольких машинах (лектор не знает), Docker Swarm (отложен на следующие пары), ДЗ.
