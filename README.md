# Momo Store  контейнеризация

Интернет-магазин: backend на Go (порт 8081), frontend на Vue.js (порт 80).
Оркестрация через Docker Compose, сборка и сканирование образов в GitHub Actions.

```
браузер ──:80──▶ nginx ──┬─▶ /momo-store/  статика
                         ├─▶ /api/         прокси ──:8081──▶ backend (Go)
                         └─▶ /healthz      200 ok
                  сеть web              сеть backend (internal)
```

Приложение stateless: хранилище фейковое, в памяти, базы данных нет, секретов нет.
Браузер обращается только к порту 80, запросы к API идут на относительный
путь `/api/`, который nginx проксирует на бэкенд по внутренней сети.

---

## Запуск

```bash
docker compose up -d --build          # prod (профиль берётся из .env)
docker compose --profile dev up -d    # dev
```

Приложение: <http://localhost/> (корень редиректит на `/momo-store/`).

```bash
docker compose ps                          # статус и healthcheck
docker compose logs -f frontend-prod       # логи
docker stats --no-stream                   # потребление и лимиты
docker compose up -d --scale backend-prod=5
docker compose down                        # остановить
docker compose down -v                     # + удалить volume с кэшем
```

Проверка:

```bash
curl -s -o /dev/null -w '%{http_code}\n' localhost/momo-store/    # 200
curl -s -o /dev/null -w '%{http_code}\n' localhost/api/products   # 200
curl -s -o /dev/null -w '%{http_code}\n' localhost:8081/health    # недоступен в prod
```

---

## Образы

| Образ | Сборка | Рантайм | Размер |
|---|---|---|---|
| backend | `golang:alpine3.24` | `alpine:3.24` | **29.6 МБ** |
| frontend | `node:16-alpine` | `nginx:1.30-alpine` | **104 МБ** |

Обе сборки multi-stage.
**Backend.** `CGO_ENABLED=0` даёт полностью статический бинарь.
`-ldflags="-s -w"` убирает отладочную информацию.
Базовый образ alpine, тк в  distroless них нет утилит для
HTTP-запроса, а `HEALTHCHECK` выполняется внутри контейнера. В финальной
стадии один `RUN` и один `COPY`, пакетный менеджер не вызывается, чистить
нечего.

**Frontend.** Node 16: webpack 4 из `package.json` на Node 17+
падает с `ERR_OSSL_EVP_UNSUPPORTED`. `npm ci` вместо `npm install` —
детерминированная установка по lock-файлу. Статика собирается в
`/usr/share/nginx/html/momo-store`, потому что `vue.config.js` задаёт
`publicPath = /momo-store/`.

**Кэширование слоёв.** В обоих Dockerfile файлы зависимостей
(`go.mod`/`go.sum`, `package.json`/`package-lock.json`) копируются до
исходников, поэтому правка кода не инвалидирует слой с зависимостями.

**`.dockerignore`** есть у обоих контекстов. Кроме мусора исключены сам
`Dockerfile` и `.dockerignore` — без этого каждая их правка ломала бы кэш
слоя `COPY . .`. В бэкенде исключён `**/mock/`, но не `**/fake/`: несмотря
на название, пакет `fake` импортируется боевым кодом.

**Пользователи.** `svc-backend` (uid 1000) и `svc-frontend` (uid 1001).
nginx слушает 8080, а не 80: порты ниже 1024 привилегированные. Требование "порт 80" закрывается публикацией `80:8080`.

**Healthcheck** объявлен в обоих Dockerfile и продублирован в compose:
`/health` у бэкенда, `/healthz` у фронтенда. Адрес `127.0.0.1`.

---

## Конфигурация

**Переменные окружения:**

| Переменная | Сервис | Назначение |
|---|---|---|
| `TZ` | оба | таймзона в логах |
| `GOMAXPROCS` | backend | потоки планировщика Go |
| `GOMEMLIMIT` | backend | мягкий потолок памяти для GC |
| `COMPOSE_PROFILES` | `.env` | профиль по умолчанию |
| `COUNT_REPLICS` | `.env` | число реплик бэкенда |

`GOMAXPROCS` и `GOMEMLIMIT` нужны потому, что процесс внутри контейнера
видит ресурсы **хоста**, а не квоту cgroup. Без них Go завёл бы потоки по
числу ядер хоста, а сборщик мусора узнал бы о лимите памяти в момент
OOM-kill. `GOMEMLIMIT` выставлен в 50 MiB — около 80 % от лимита в 64 МБ.

**Build-аргументы:**

| Аргумент | Образ | По умолчанию |
|---|---|---|
| `VUE_APP_API_URL` | frontend | `/api` |

Это именно build-time аргумент: Vue CLI подставляет значение текстом в
бандл при сборке, в рантайме переменная не читается.

---

## Docker Compose

**Профили.**

| | `dev` | `prod` |
|---|---|---|
| Порт 8081 на хосте | открыт | закрыт |
| `read_only` | нет | да |
| Лимиты CPU/RAM | нет | 0.5 CPU / 64 МБ |
| `cap_drop` | нет | ALL |
| Реплики бэкенда | 1 | из `.env` |

Все сервисы объявлены с `profiles:`, поэтому в `.env` прописан
`COMPOSE_PROFILES=prod`, голый запуск поднимает продакшн-вариант.

Сервисы называются `backend-dev`/`backend-prod`, но оба получают сетевой
алиас `backend`, поэтому `nginx.conf` одинаков для обоих профилей.
Общая часть вынесена в YAML-якоря.

**Сети.** `web` публичная, `backend` с `internal: true`, без маршрута
наружу. В prod бэкенд подключён только к внутренней сети и недоступен
с хоста. В dev он дополнительно подключён к `web` для простоты отладки.

**Volumes.** Named volume `nginx-cache` смонтирован в `/var/cache/proxy` —
там nginx хранит дисковый кэш ответов бэкенда. Это единственные данные в
проекте, которые рационально сохранять.

Кэш намеренно вынесен за пределы `/var/cache/nginx`: тот путь смонтирован
как tmpfs.

Каталог кэша создаётся в Dockerfile с `chown` до `USER`: named volume
при первом создании наследует владельца из образа, и без этого он оказался
бы root'овым, а nginx работает под uid 1001.

`/api/auth/whoami` вынесен в отдельный `location` мимо кэша, ответ зависит
от пользователя.

**Лимиты** заданы по факту замера: фронтенд ест 9.4 МБ, бэкенд 5.1 МБ при
лимите 64 МБ.

**Restart policy** `unless-stopped` у всех сервисов. Compose не
перезапускает контейнер по статусу `unhealthy`: restart-политики реагируют
только на завершение процесса. Healthcheck здесь нужен для
`depends_on: condition: service_healthy`.

---

## Масштабирование

```bash
docker compose up -d --scale backend-prod=3
```

Масштабируется только бэкенд: он stateless (липкие сессии не нужны, любой
запрос обслужит любая реплика) и не публикует порт на хост. Фронтенд
публикует порт 80, вторая реплика не поднялась бы.

Балансировку обеспечивает встроенный DNS Docker, но для этого пришлось
заставить nginx перечитывать записи:

```nginx
resolver 127.0.0.11 valid=10s ipv6=off;
set $backend http://backend:8081;
rewrite ^/api/(.*)$ /$1 break;
proxy_pass $backend;
```
Замер: 30 запросов на некэшируемый эндпоинт при трёх репликах
распределились как **9 / 8 / 13**.

---

## Безопасность

- Оба контейнера работают под непривилегированными пользователями.
- Наружу открыт только порт 80.
- В рантайм-образах не выполняется ни одной команды пакетного менеджера.
- В prod: `read_only: true`, `cap_drop: [ALL]`,
  `security_opt: [no-new-privileges:true]`.

Бэкенду не понадобилось ни одного writable-пути.
Фронтенду выданы две tmpfs (`/run` и `/var/cache/nginx`) с явными
`uid`/`gid`: по умолчанию tmpfs монтируется от root. 

`cap_drop: [ALL]` безболезненно, потому что оба процесса работают под
непривилегированными пользователями на портах выше 1024 — даже
`NET_BIND_SERVICE` не требуется.

**Сканирование.** Джоба `scan-vulnerabilitues` прогоняет Trivy 

| Образ | База | HIGH | CRITICAL |
|---|---|---|---|
| backend | alpine 3.24.1 | 2 | 0 |
| frontend | alpine 3.24.1 | 7 | 0 |

Две HIGH в бэкенде, `CVE-2026-14456` в OpenSSL. Бинарь собран с
`CGO_ENABLED=0` и с OpenSSL не линкуется, библиотеки просто присутствуют
в базовом образе.

**Docker secrets** не использовался, так как у приложения нет секретов.

---

## CI/CD

`.github/workflows/deploy.yaml`. Исходные джобы не изменялись, добавлена
`scan-and-push-tagged`, матрица на два сервиса:

```
checkout → buildx → build (push:false, load:true)
        → Trivy report 
```


---

## Структура

```
backend/     Dockerfile, .dockerignore, исходники Go
frontend/    Dockerfile, .dockerignore, nginx.conf, исходники Vue
.github/workflows/deploy.yaml
docker-compose.yml
.env
```
