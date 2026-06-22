# Momo-store: контейнеризация Go + Vue

Учебный проект по дисциплине «Docker-контейнеризация и хранение данных».
Приложение **momo-store** («Пельменная №2»): бэкенд на Go, фронтенд на Vue.js.

## Запуск

```bash
docker compose up --build -d
docker compose ps
```

- Фронтенд: <http://localhost> (порт 80)
- API: <http://localhost/api/products> (проксируется на бэкенд)
- Остановка: `docker compose down`

## Образы

| Образ           | Базовый образ финала                       | Размер   |
|-----------------|--------------------------------------------|----------|
| `momo-backend`  | `alpine:3.19`                              | ~26 МБ   |
| `momo-frontend` | `nginxinc/nginx-unprivileged:1.27-alpine`  | ~76 МБ   |

**Как добились такого размера:**
- **Multi-stage сборки.** В финал бэкенда попадает только скомпилированный бинарник (без Go-компилятора), в финал фронта — только статика из `dist` (без Node и `node_modules`).
- **Лёгкие базовые образы** на alpine.
- **Статическая сборка Go** (`CGO_ENABLED=0`, `-ldflags="-s -w"`) — без отладочных символов.
- **Кэширование слоёв:** сначала копируются манифесты зависимостей (`go.mod`, `package.json`) и ставятся зависимости, потом исходники — пересборка при правке кода идёт за секунды.
- **`.dockerignore`** исключает `.git`, `node_modules`, `dist`, тесты, логи.

## Конфигурируемость

- **Бэкенд** слушает порт 8081.
- **Фронтенд** конфигурируется build-arg `VUE_APP_API_URL` (по умолчанию `/api`). Особенность Vue CLI — `VUE_APP_*` подставляются в бандл на этапе сборки:
```yaml
  frontend:
    build:
      args:
        VUE_APP_API_URL: /api
```
- **nginx фронтенда** отдаёт SPA с fallback на `index.html` (для history-режима роутера) и проксирует `/api/*` на `backend:8081`.

## Compose

- Изолированная bridge-сеть `momo-net`. Бэкенд наружу не публикуется (`expose`, без `ports`), снаружи доступен только фронт на 80.
- `depends_on: condition: service_healthy` — фронт стартует после healthcheck бэкенда.
- **Healthcheck'и** у обоих сервисов.
- **Лимиты ресурсов** (`deploy.resources.limits`): backend 0.5 CPU / 256 MB, frontend 0.5 CPU / 128 MB.
- Политика перезапуска `restart: unless-stopped`.

## Масштабирование и балансировка

```bash
docker compose up -d --scale backend=3
```

Балансировка через nginx фронтенда: подключён DNS Docker (`resolver 127.0.0.11`), имя `backend` подставляется через переменную, поэтому nginx **резолвит его в рантайме на каждый запрос** и раскидывает запросы по всем репликам. У бэкенда нет проброса портов на хост — реплики не конфликтуют за порт.

Проверка:
```bash
for i in $(seq 1 6); do curl -s -o /dev/null -w "%{http_code}\n" http://localhost/api/products; done
docker compose logs backend | tail -30   # запросы видны в разных репликах
```

Возврат к одной реплике — приложение продолжает работать:
```bash
docker compose up -d --scale backend=1
```

## Безопасность

- **Непривилегированные пользователи.** Бэкенд работает под `app`, фронт — на `nginx-unprivileged` (uid 101), nginx внутри контейнера слушает 8080.
- **Минимальные финальные образы** на alpine: ни компиляторов, ни шеллов разработки, ни кэшей сборки.
- **Только нужные порты:** наружу опубликован только фронт (80). Бэкенд внутренний.
- **Никаких секретов в образах и коде.** Учётные данные Docker Hub берутся из GitHub Secrets (`DOCKER_USER`, `DOCKER_PASSWORD`).
- **Лимиты CPU/памяти**, **изоляция в bridge-сети** `momo-net`.

### Сканирование образов (Trivy)

В пайплайн добавлена джоба `trivy_scan`, которая сканирует оба образа на уязвимости `HIGH`/`CRITICAL`. Существующие шаги пайплайна не изменены.

Локально:
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image --severity CRITICAL,HIGH --ignore-unfixed momo-backend:latest
```

Используются флаги `--ignore-unfixed` (только уязвимости с доступным фиксом) и `--severity CRITICAL,HIGH`. В CI Trivy запускается с `exit-code: 0`: пайплайн не падает из-за уязвимостей в третьесторонних зависимостях, но отчёт всегда виден в логах джобы.

## CI/CD

`.github/workflows/deploy.yaml` запускается на push в `main`:

1. `build_and_push_to_docker_hub` — сборка и публикация образов в Docker Hub.
2. `run-with-docker-compose` — запуск приложения на раннере через compose.
3. `trivy_scan` — сканирование собранных образов (добавлено сверху, существующие шаги не тронуты).

Требуемые секреты репозитория: `DOCKER_USER`, `DOCKER_PASSWORD` (Docker Hub access token).
