# Модуль 6. Финальная интеграция — условие

## Задание

Собрать всё вместе. Взять HTTP-сервер из Проекта 4 и заменить файловое хранилище на `TaskRepository` из модулей 4-5. Это **финальная сборка** Проекта 5.

## Что должна делать программа

```bash
# Запуск контейнера
$ docker run -d --name todo-pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -v todo-pg-data:/var/lib/postgresql/data postgres:16
$ sleep 3

# Применение миграций
$ export PGPASSWORD=secret
$ for f in migrations/*.sql; do psql -h localhost -U postgres -d postgres -f "$f"; done

# Запуск сервера
$ DATABASE_URL="postgres://postgres:secret@localhost:5432/postgres?sslmode=disable" go run .
2026/04/13 12:00:00 Подключение к PostgreSQL установлено
2026/04/13 12:00:00 Сервер запущен на http://localhost:8080
```

В другом терминале:

```bash
$ curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" -d '{"title":"Купить молоко"}'
{"id":1,"title":"Купить молоко","done":false}

$ curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" -d '{"title":"Позвонить маме"}'
{"id":2,"title":"Позвонить маме","done":false}

$ curl http://localhost:8080/tasks
[{"id":1,"title":"Купить молоко","done":false},{"id":2,"title":"Позвонить маме","done":false}]

$ curl -X PUT http://localhost:8080/tasks/1 -H "Content-Type: application/json" -d '{"title":"Купить молоко","done":true}'
{"id":1,"title":"Купить молоко","done":true}

$ curl -X DELETE http://localhost:8080/tasks/2
# (пустой ответ, 204)

$ curl http://localhost:8080/tasks
[{"id":1,"title":"Купить молоко","done":true}]

# Останови сервер (Ctrl+C), запусти снова
$ DATABASE_URL=... go run .
$ curl http://localhost:8080/tasks
[{"id":1,"title":"Купить молоко","done":true}]
# Данные на месте!
```

## Требования

1. Финальная структура проекта:
   ```
   005-todo-postgres/
   ├── go.mod
   ├── go.sum
   ├── main.go
   ├── server.go
   ├── repository.go
   ├── task.go
   └── migrations/
       ├── 001_create_tasks_table.sql
       ├── 002_index_done.sql
       └── 003_check_title_not_empty.sql
   ```

2. **Все файлы — `package main`**.

3. В `task.go`:
   ```go
   type Task struct {
       ID    int64  `json:"id"`
       Title string `json:"title"`
       Done  bool   `json:"done"`
   }
   ```

4. В `repository.go` — `TaskRepository` со всеми пятью методами (`GetAll`, `GetByID`, `Add`, `Update`, `Remove`) из модуля 4.

5. **`TaskRepository` НЕ имеет `sync.Mutex`**. Никаких `Lock`/`Unlock`. Это — момент удаления.

6. В `server.go` — тип `Server` с полем `tasks *TaskRepository` и пять обработчиков:
   - `listTasks` (GET /tasks)
   - `getTask` (GET /tasks/{id})
   - `createTask` (POST /tasks)
   - `updateTask` (PUT /tasks/{id})
   - `deleteTask` (DELETE /tasks/{id})

7. Каждый обработчик обрабатывает **три типа исходов** от репозитория:
   - **Успех** → 200/201/204 + данные
   - **Не найдено** (`errors.Is(err, sql.ErrNoRows)` или твоя ошибка) → 404
   - **Техническая ошибка** → 500 + лог + клиенту общее сообщение

8. **DTO для создания и обновления** — те же, что в Проекте 4 (отдельная структура с одним/двумя полями, не сам `Task`).

9. **Валидация ввода** — пустой `title` → 400.

10. В `main.go`:
    - `DATABASE_URL` читается из `os.Getenv`
    - `sql.Open("pgx", dsn)` + `defer db.Close()`
    - Настройка пула: `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`
    - `db.Ping()` с обработкой ошибки
    - Создание `TaskRepository` и `Server`
    - Регистрация обработчиков с path patterns
    - `http.ListenAndServe(":8080", nil)`

11. **Никаких `Save`/`Load`** в коде.

12. **Никаких файлов с данными** в проекте.

13. **Полный сценарий** работает: создание, чтение, обновление, удаление через `curl`.

14. **После перезапуска** сервера данные на месте.

## Шаги

1. Создай папку `005-todo-postgres/app/` (или используй корень — твой выбор)
2. Инициализируй модуль: `go mod init todo-postgres`
3. Установи драйвер: `go get github.com/jackc/pgx/v5/stdlib`
4. Создай `task.go` с типом `Task`
5. Создай `repository.go` с `TaskRepository` и пятью методами (используй код из модулей 4-5)
6. Создай `server.go` с типом `Server` и обработчиками (адаптируй из Проекта 4)
7. Создай `main.go` с инициализацией БД и сервера
8. Помести миграции в `migrations/`
9. Применени миграции
10. Запусти сервер
11. Прогон полного сценария через `curl`
12. Останови, запусти снова, проверь, что данные на месте

## Эксперименты

1. **Запусти два экземпляра на разных портах** (поправь код, чтобы порт читался из `os.Getenv("PORT")`):
   ```bash
   PORT=8080 DATABASE_URL=... go run . &
   PORT=8081 DATABASE_URL=... go run . &
   ```
   Создай задачу через первый, запроси через второй. Видна? Конечно — БД одна, оба сервера её читают. Это то, что в Проекте 4 ломалось.

2. **Запусти нагрузочный тест из Проекта 4:**
   ```bash
   seq 1 100 | xargs -n1 -P20 -I{} curl -s -X POST http://localhost:8080/tasks \
       -H "Content-Type: application/json" \
       -d '{"title":"task {}"}'
   ```
   Проверь, что в БД ровно 100 новых задач (плюс старые), все с уникальными id. **Никаких race conditions** — БД сама всё корректно обрабатывает.

3. **Останови БД во время работы сервера:**
   ```bash
   docker stop todo-pg
   curl http://localhost:8080/tasks
   ```
   Что произошло? Получил 500 с понятной ошибкой в логе сервера. Это правильно: сервер не падает, а возвращает ошибку клиенту. Запусти БД (`docker start todo-pg`) — следующий запрос **сам** установит новое соединение из пула.

4. **Попробуй передать невалидный JSON в POST:**
   ```bash
   curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" -d 'не json'
   ```
   400, как в Проекте 4. Валидация на уровне HTTP.

5. **Попробуй передать пустой title в POST:**
   ```bash
   curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" -d '{"title":""}'
   ```
   Должно быть 400 — сначала валидация в Go-коде, потом (если убрать валидацию) — 500 от CHECK constraint в БД. Лучше валидация на двух уровнях: сначала Go (для понятного сообщения клиенту), потом БД (для гарантии целостности).

6. **Проверь работу контекстов** (если ты добавил `*Context` методы):
   - Открой долгий запрос (через `time.Sleep` в тестовом эндпоинте)
   - Прерви клиент через Ctrl+C
   - Лог сервера должен показать, что запрос отменён через контекст

7. **Запусти `EXPLAIN ANALYZE` для частого запроса:**
   ```sql
   EXPLAIN ANALYZE SELECT id, title, done FROM tasks WHERE id = 1;
   ```
   Должен использовать `Index Scan using tasks_pkey`. Это работа PRIMARY KEY — поиск по id мгновенный.

8. **Сравни количество строк кода до и после.** Открой Проект 4 и Проект 5, посчитай строки в `tasklist.go`/`repository.go`. У многих после перехода на БД код становится **короче**, потому что не надо вручную управлять состоянием и мьютексами.

## Что показать Claude

Покажи Claude свой `main.go` после интеграции и спроси: «правильно ли я разделяю ответственности? Что бы я разделил по пакетам, если бы проект рос?». Это вопрос на будущее. Возможные ответы:

- **`internal/storage/`** — `repository.go` и миграции
- **`internal/api/`** — `server.go` и обработчики
- **`internal/domain/`** — `task.go` и связанная бизнес-логика
- **`cmd/server/main.go`** — точка входа

Это структура, которую ты увидишь в Проектах 6-7. Сейчас нет смысла это делать — в маленьком проекте лишние пакеты только усложняют. Но **держи это в голове**, чтобы знать, как код будет расти.

Запиши себе на будущее: **архитектура — это не про то, как красиво написать с нуля**, а про то, **как код переживает рост**. Хорошая архитектура не «правильная сейчас», а **легко изменяемая в будущем**.
