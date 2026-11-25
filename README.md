# Aquarium Server

Асинхронный серверный бэкенд трекера задач для команды, разработанный на **C++20** с использованием фреймворка **userver** (v3.2), протокола **gRPC** и базы данных **PostgreSQL 16**.

---

## Основные возможности

- **Управление пользователями (`UserStoreService`)**:
  - Регистрация по инвайт-ключу (`AddUser`) с генерацией UUID-токена.
  - Получение списка и профиля активных пользователей (`GetAll`, `GetUser`) без возврата токенов авторизации.
  - Обновление имени (`UpdateUser`) и мягкое удаление (`DeleteUser` — установка `active = FALSE` в `user_schema.tokens`).
- **Управление задачами и ревью (`TaskStoreService`)**:
  - Полный CRUD по задачам (`CreateTask`, `GetTask`, `GetTasks`, `UpdateTask`, `UpdateTaskStatus`, `DeleteTask`).
  - Фильтрация (`status`, `authorId`, `workerId`) и SQL-пагинация в `GetTasks` (`limit` по умолчанию `50`, максимум `100`, `offset`).
  - Статусы задач: `PLAN`, `ACTIVE`, `IN_REVIEW`, `COMPLETE`, `VERIFIED`, `ARCHIVE`.
  - Назначение и снятие исполнителей (`AssignWorker`, `RemoveWorker`) с проверкой, что пользователь активен.
  - Комментарии (`AddComment`, `GetComments`) и атомарный возврат задачи на доработку (`RejectTask` — переводит статус в `ACTIVE` и добавляет комментарий `COMMENT_REVISION_REASON`).
  - Стриминг событий в реальном времени (`SubscribeEvents` → `stream TaskEvent`) через `userver::concurrent::AsyncEventChannel`.
- **Безопасность, кэширование и авторизация**:
  - In-memory кэш пользователей и токенов `auth-pg-cache` (`userver::components::PostgreCache`, инкрементальное обновление каждые `10s` и `InvalidateAsync` при мутациях).
  - `AuthenticationMiddleware` с постоянновременным сравнением ключей (`ConstantTimeCompare`) и проверкой активности пользователя.
  - Ролевой контроль доступа `AccessControl` (`admin`, автор задачи, проверяющий `checkerId`, исполнитель `workerIds`).
  - Валидация входных данных (название задачи до `255` символов, описание до `65536` байт, комментарий/причина отклонения до `16384` байт, имя пользователя до `100` символов) и проверка стойкости секретных ключей при старте.
- **Healthcheck (`UtilsService`)**:
  - RPC `Health` для проверки готовности сервиса и связности с БД через внутренний gRPC-клиент.

---

## Быстрый старт с Docker (Docker Compose)

Для запуска сервиса и базы данных достаточно иметь установленные **Docker** и **Docker Compose**.

### 1. Клонирование репозитория

```bash
git clone https://github.com/Aukvary/Aquarium-server.git
cd Aquarium-server
```

### 2. (Опционально) Настройка окружения

Если нужно изменить стандартные порты, параметры БД или секретные ключи, создайте `.env` на основе шаблона:

```bash
cp .env.example .env
```

По умолчанию сервис работает из коробки со значениями из `docker-compose.yml`.

### 3. Запуск контейнеров

```bash
docker compose up -d --build
```

### Что происходит при запуске

1. **Контейнер базы данных (`aquarium-db`)**:
   - Запускает PostgreSQL 16 (`postgres:16-alpine`).
   - При первом старте выполняет `scripts/init-db.sh`, применяя миграции в нужном порядке: `sql/users.sql` → `sql/Tasks.sql` → `sql/comments.sql`.
   - Проверяет готовность через `pg_isready`.
2. **Контейнер приложения (`aquarium-app`)**:
   - Собирается в Release-режиме на базе `ghcr.io/userver-framework/ubuntu-24.04-userver:latest` с `-DBUILD_TESTING=OFF`.
   - Проверяет ключи `INTERNAL_KEY` и `INVITE_KEY`, подключается к БД и прогревает `auth-pg-cache`.
   - Запускает gRPC-сервер (`8081`) и системный HTTP-сервер userver (`8080`).

---

## Сетевые порты

| Сервис | Порт по умолчанию | Описание |
|---|---|---|
| **gRPC API** | `8081` | Основной gRPC-интерфейс (`UserStoreService`, `TaskStoreService`, `UtilsService`) |
| **HTTP Listener** | `8080` | Системный HTTP-порт userver (мониторинг / метрики / testsuite) |
| **PostgreSQL** | `5421` | Порт БД на хосте (привязан к `127.0.0.1`, внутри Docker — `5432`) |

---

## Переменные окружения (`.env` и `configs/config_vars.yaml`)

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `POSTGRES_USER` | `admin` | Пользователь PostgreSQL |
| `POSTGRES_PASSWORD` | `password` | Пароль пользователя PostgreSQL |
| `POSTGRES_DB` | `aquarium` | Имя базы данных |
| `POSTGRES_PORT` | `5421` | Внешний порт PostgreSQL на `127.0.0.1` |
| `GRPC_PORT` | `8081` | Внешний порт gRPC-сервиса |
| `HTTP_PORT` | `8080` | Внешний системный HTTP-порт |
| `INTERNAL_KEY` | `aquarium_internal_secret_key_minimum_32_chars` | Ключ для внутренних межсервисных вызовов |
| `INVITE_KEY` | `aquarium_invite_secret_key_minimum_32_chars` | Ключ инвайта для регистрации через `UserStoreService/AddUser` |
| `PG_CONNECTION` | `postgresql://admin:password@127.0.0.1:5432/aquarium` | Строка подключения к PostgreSQL (в Docker Compose задаётся как `@db:5432`) |
| `USER_STORE_CLIENT_ENDPOINT` | `127.0.0.1:8081` | Адрес gRPC-сервера для внутреннего клиента `user-store-client` |
| `AQUARIUM_TEST_MODE` | `false` (`0`) | При `1` или `true` отключает строгую проверку длины/сложности ключей при запуске |

> **Требования безопасности к ключам (production-режим)**:
> При `test-mode: false` (по умолчанию) `INTERNAL_KEY` и `INVITE_KEY` должны быть не короче **32 символов**, содержать не менее **4 уникальных символов** и не совпадать с известными слабыми значениями (`zxc`, `secret_invite_key_value`, `password`, `admin` и т.п.).
> Доступ по `INTERNAL_KEY` разрешён только с loopback/локальных адресов (`127.x.x.x`, `::1`, `unix:`, `inproc`); с внешних адресов возвращается `PERMISSION_DENIED`.

---

## Локальная сборка и разработка

### Сборка через CMake и Ninja

Если `userver` не установлен в системе, `CMakeLists.txt` автоматически загрузит `userver v3.2` через `FetchContent`.

```bash
# 1. Запустить PostgreSQL (если разрабатываете локально)
POSTGRES_PORT=5432 docker compose up -d db

# 2. Собрать проект вместе с тестами (или -DBUILD_TESTING=OFF для сборки без тестов)
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j$(nproc)
```

Также для локальной инициализации БД (`127.0.0.1:5432`) и Debug-сборки можно использовать `./build.sh` (ожидает PostgreSQL на порту `5432`).

### Запуск собранного сервера

```bash
USERVER_ENABLE_STACK_USAGE_MONITOR=0 \
./build/aquarium --config configs/static_config.yaml --config-vars configs/config_vars.yaml
```

---

## Полезные команды Docker

- **Просмотр логов приложения**:
  ```bash
  docker compose logs -f app
  ```
- **Просмотр логов базы данных**:
  ```bash
  docker compose logs -f db
  ```
- **Остановка контейнеров**:
  ```bash
  docker compose down
  ```
- **Остановка и полный сброс данных БД (удаление volume)**:
  ```bash
  docker compose down -v
  ```

---

## Тестирование

### C++ Unit и Coroutine тесты (`userver::utest`)

```bash
cmake --build build --target aquarium-unittest
ctest --test-dir build --output-on-failure
```

### Функциональные интеграционные тесты (`pytest` + `pytest-userver`)

Интеграционные тесты в `tests/` используют сгенерированные proto-модули из `build/proto` и тестовые переменные из `configs/config_vars.test.yaml`:

```bash
pytest tests/
```

---

## Проверка работоспособности и примеры gRPC (`grpcurl`)

### 1. Проверка Healthcheck (`UtilsService/Health`)

На `health-service` middleware аутентификации отключено, поэтому вызов не требует токена:

```bash
grpcurl -plaintext localhost:8081 aquarium.api.UtilsService/Health
```

### 2. Регистрация пользователя (`UserStoreService/AddUser`)

Требует заголовок `invite-key: Invite <INVITE_KEY>`. В ответе возвращается `id`, `name` и сгенерированный токен `key`:

```bash
grpcurl -plaintext \
  -H "invite-key: Invite aquarium_invite_secret_key_minimum_32_chars" \
  -d '{"name": "Alice"}' \
  localhost:8081 aquarium.api.UserStoreService/AddUser
```

### 3. Вызов методов от имени пользователя

Для всех остальных RPC (`UserStoreService` и `TaskStoreService`) передавайте токен пользователя в заголовке `authorization: Bearer <key>`:

```bash
# Список активных пользователей
grpcurl -plaintext \
  -H "authorization: Bearer <user_key>" \
  -d '{}' \
  localhost:8081 aquarium.api.UserStoreService/GetAll

# Создание задачи
grpcurl -plaintext \
  -H "authorization: Bearer <user_key>" \
  -d '{"title": "Setup server", "description": "Verify gRPC API", "authorId": 1, "status": "PLAN"}' \
  localhost:8081 aquarium.api.TaskStoreService/CreateTask
```

### Справка по gRPC-сервисам (`proto/`)

- **`aquarium.api.UserStoreService`**: `GetAll`, `GetUser`, `AddUser`, `UpdateUser`, `DeleteUser`
- **`aquarium.api.TaskStoreService`**: `CreateTask`, `GetTask`, `GetTasks`, `UpdateTask`, `UpdateTaskStatus`, `AssignWorker`, `RemoveWorker`, `DeleteTask`, `AddComment`, `GetComments`, `RejectTask`, `SubscribeEvents`
- **`aquarium.api.UtilsService`**: `Health`

---

## Структура проекта

```text
Aquarium-server/
├── CMakeLists.txt              # Конфигурация CMake (userver, proto, aquarium, aquarium-unittest)
├── Dockerfile                  # Release-сборка сервиса на базе ubuntu-24.04-userver
├── docker-compose.yml          # Оркестрация PostgreSQL 16 и приложения
├── build.sh                    # Скрипт инициализации локальной БД и Debug-сборки
├── .env.example                # Шаблон переменных окружения
├── configs/
│   ├── static_config.yaml      # Топология компонентов userver
│   ├── config_vars.yaml        # Переменные конфигурации (env + fallback)
│   └── config_vars.test.yaml   # Переменные конфигурации для pytest-тестов
├── proto/
│   ├── UserStoreService.proto  # Контракт управления пользователями
│   ├── TaskStoreService.proto  # Контракт задач, комментариев и TaskEvent-стрима
│   └── UtilsService.proto      # Контракт Healthcheck
├── scripts/
│   └── init-db.sh              # Инициализация схем PostgreSQL в Docker
├── sql/
│   ├── aquarium.sql            # Создание БД aquarium
│   ├── users.sql               # Схема user_schema.tokens и триггер updated
│   ├── Tasks.sql               # Схема task_schema (tasks, task_workers, task_cache_view, индексы и триггеры)
│   └── comments.sql            # Таблица task_schema.task_comments и индекс
├── src/
│   ├── main.cpp                # Регистрация компонентов и запуск DaemonMain
│   ├── Users.hpp / Users/      # AuthCache, AuthenticationMiddleware, UserStoreService
│   ├── TaskStore.hpp / Tasks/  # TaskStoreService и брокер событий SubscribeEvents
│   ├── AccessControl.hpp       # Правила ролевой авторизации
│   └── Utils.hpp / Utils/      # UtilsService (Health) и реализация AccessControl
└── tests/
    ├── conftest.py             # Фикстуры pytest-userver и PostgreSQL
    ├── test_users.py           # Интеграционные тесты UserStoreService
    ├── test_tasks.py           # Интеграционные тесты TaskStoreService
    └── unittest/               # C++ unit/coroutine тесты (userver::utest)
```
