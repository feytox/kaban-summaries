---
date: 2026-10-02
ingest: fast
links:
  - https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz
---
# L03. Docker на практике: dive, хранение данных, Compose

Продолжение [[devops-L02-docker-образы-и-слои|L02]]. По лектору, Dockerfile на работе правят редко: в проекте он уже есть, меняют его при смене версии языка или способа деплоя. Важнее понимать, как устроены образы, потому что дальше будет оркестрация (Kubernetes). ([04:20](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=260))

## dive: разбор образа по слоям

**dive** - утилита, которая показывает слои образа, команду каждого и снапшот файловой системы после него. ([08:31](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=511))

- **efficiency** - доля места без потерь: файл, который трогали в нескольких слоях (добавили и удалили), считается потраченным впустую ([12:03](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=723))
	- плохой Dockerfile с пары дал 77%, хороший с установкой и чисткой в одном слое - почти 100%
- в CI dive запускают с порогами (размер, потери в %): неоптимальный Dockerfile ловится автоматически, как падение покрытия тестами ([15:02](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=902))
- пороги задаёшь сам; под строгие правила лектора не проходит даже стандартный образ Python 3.14

```bash
dive myapp:1.0          # интерактивно
CI=true dive myapp:1.0  # проверка по правилам из .dive-ci, exit code != 0 при нарушении
```

## Размер образа против времени сборки

([21:26](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1286))
- базовые образы (`slim`, обычный и др.) различаются набором заранее установленных библиотек и бинарников
- простому веб-серверу хватит `slim`; с NumPy, ML и другими C-зависимостями образ всё равно дорастёт до обычного, а сборка заметно удлинится
- **размер и время сборки - trade-off**: сборка идёт почти на каждый коммит, поэтому решают по бенчмарку холодной сборки и dive ([24:32](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1472))
- **лектор делает упор на время сборки**: 100 разработчиков, ждущих пайплайн, стоят дороже дисков на раннерах ([39:05](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2345))
	- на приватные раннеры CI заранее кладут общие базовые образы, чтобы джобы не скачивали их заново
- в Точке, где работает лектор, используют обычные образы, не `slim`
- multi-stage для Python (оставить только интерпретатор и код) сжал образ с 400+ до 136 МБ, но лектор называет это "супер странной оптимизацией" ([20:04](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1204))
- в Точке первая стадия только задаёт env для пакетного менеджера, адреса registry и пути к сертификатам; её слои не попадают в итоговый образ ([29:01](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1741))

## Один образ на все окружения

([32:03](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=1923))

| Подход               | Плюс                                                          | Минус                                        |
| -------------------- | ------------------------------------------------------------- | -------------------------------------------- |
| один образ на всё    | тесты, тестировщик и прод видят один и тот же код и окружение | тесты и их зависимости едут в прод           |
| образ на каждый этап | в проде ничего лишнего (нет `tests/`, mypy, ruff)             | прод запускает не тот образ, что тестировали |

- **лектор рекомендует один образ на всё**: его собирают один раз и прокатывают через тесты, тестовые контуры и прод (ср. фактор 5 в [[sre-P01-devops-sre-12-факторов#Код и сборка|SRE P01]])
- dev-зависимости (линтеры) в продовый образ не ставят в обоих подходах

## Хранение данных на практике

([41:25](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2485))

```bash
docker volume create st-data
docker run --rm -v st-data:/data alpine:3.24 sh -c 'echo "данные в томе" > /data/message.txt'
docker run --rm -v st-data:/data alpine:3.24 cat /data/message.txt   # данные пережили контейнер
docker run --rm -v "$PWD/hostdir:/data" alpine:3.24 ls /data          # bind mount: директория хоста
```

Команды восстановлены по рассказу.

- **volume** - объект под управлением Docker: в его файлы на хосте не лезут, данные достают через подключённый контейнер ([46:43](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2803))
- один volume можно подключить к нескольким контейнерам, но общий доступ к файлам грозит гонками, это не best practice ([45:01](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2701))
- **bind mount** нужен для локальной разработки: watch-режим фронтенда или Django перезапускает приложение при правке кода без пересборки образа
- **частая ошибка**: контейнер без `USER` создаёт файлы в bind mount от root, и на хосте их не прочитать без sudo ([49:08](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=2948))
- **tmpfs** хранит секреты, полученные из Vault и подобных систем, а не через env в Dockerfile ([54:24](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3264))
- в `-v` том и mount отличаются только первой частью; лектор обещал уточнить после пары ([51:02](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3062))
	- (вне конспекта) имя (`st-data:/data`) - volume, путь на хосте (`/abs/path:/data`) - bind mount ([docs](https://docs.docker.com/engine/storage/bind-mounts/))

## Docker Compose

**Docker Compose** - оркестрация "на минималках": один файл описывает целевое состояние всех контейнеров вместо длинных `docker run`. По лектору, в проде не используется, только для локального запуска проекта; в проде Kubernetes. ([56:39](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3399))

```yaml
services:
  app:
    build: .                          # или image: готовый образ
    environment:
      REDIS_HOST: cache               # хост = имя сервиса, DNS внутри сети compose
    depends_on:
      db: {condition: service_healthy}   # стартовать после healthcheck БД
      cache: {condition: service_healthy}
    networks: [back, front]
    restart: unless-stopped           # перезапуск при падении
    deploy:
      resources: {limits: {cpus: "0.5", memory: 256M}}  # cgroups
  db:
    image: postgres:18
    healthcheck:
      test: ["CMD", "pg_isready"]     # exit code 0 = готов
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s               # время на первый запуск
    volumes: [pgdata:/var/lib/postgresql/data]  # volume только у БД
    networks: [back]                  # порт наружу не открыт
  cache:
    image: redis:8
    healthcheck: {test: ["CMD", "redis-cli", "ping"]}
    networks: [back]
  web:
    image: nginx:1.29
    ports: ["80:80"]
    networks: [front]                 # из nginx БД недоступна, app доступно
volumes: {pgdata: {}}
networks: {back: {}, front: {}}
```

Файл мой, по описанию файла с пары. ([01:00:09](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3609))

```bash
docker compose up -d --build   # в фоне, пересобрать изменившееся
docker compose logs -f app     # имя сервиса, не контейнера
docker compose exec app sh
```

- сеть compose изолирована от хоста, сервисы находят друг друга по DNS-именам, IP каждый раз новые ([01:01:28](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3688))
- приложение на хосте, а БД в compose: тогда порт БД пробрасывают наружу
- Redis в примере считается чистым кэшем, поэтому volume ему не дают ([01:06:17](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=3977))
- разнести compose по нескольким VM, по лектору, вряд ли можно, нужно смотреть документацию; про Docker Swarm он не помнит, жив ли тот ([01:11:02](https://fiit.ktalk.ru/recordings/OfDNdQlybbeRYjxjAMgz?t=4262))

> [!note]- Пропущено
> отзывы о прошлом ДЗ, показ рабочего Dockerfile лектора, ДЗ.
