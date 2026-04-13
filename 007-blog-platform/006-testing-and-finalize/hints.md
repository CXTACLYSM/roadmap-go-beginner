# Модуль 6. Тесты и финализация — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Как называются файлы тестов в Go?
2. Какая сигнатура у тестовой функции?
3. Какой пакет даёт фейковые `http.ResponseWriter` и `*http.Request`?
4. Какая библиотека запускает Docker-контейнеры из тестов?
5. Какой метод `http.Server` останавливает сервер, дождавшись активных запросов?

## Уровень 2: про структуру тестового файла

Тесты лежат **рядом с кодом**, в том же пакете:

```
internal/
├── auth/
│   ├── service.go
│   └── service_test.go     ← тесты здесь
├── storage/
│   ├── user.go
│   └── user_test.go
```

Файл `service_test.go` начинается с `package auth` (тот же пакет, что и `service.go`). Это **white-box** тестирование — тест может обращаться к приватным функциям.

Альтернатива — `package auth_test` — black-box, только публичный API. Для нашего проекта не критично, можешь использовать любой.

## Уровень 3: про самый простой тест

Минимум:

```go
package auth

import "testing"

func TestHashPassword(t *testing.T) {
    svc := NewService("secret")
    hash, err := svc.HashPassword("password123")
    if err != nil {
        t.Fatalf("HashPassword: %v", err)
    }
    if len(hash) == 0 {
        t.Error("hash is empty")
    }
}
```

Запуск:

```bash
go test ./internal/auth/
```

Если вывод `PASS` — тест прошёл. Если `FAIL` — увидишь подробности.

## Уровень 4: про table-driven tests

Идиома Go:

```go
func TestSomething(t *testing.T) {
    cases := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid", "foo", false},
        {"empty", "", true},
        {"too long", strings.Repeat("a", 1000), true},
    }

    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            err := doSomething(tc.input)
            if (err != nil) != tc.wantErr {
                t.Errorf("wantErr=%v, got %v", tc.wantErr, err)
            }
        })
    }
}
```

Это один из главных паттернов. Научись писать его без заглядывания в документацию.

## Уровень 5: про httptest

```go
import (
    "bytes"
    "net/http"
    "net/http/httptest"
)

req := httptest.NewRequest(
    http.MethodPost,
    "/register",
    bytes.NewReader([]byte(`{"email":"test@example.com","password":"secret"}`)),
)
req.Header.Set("Content-Type", "application/json")

rec := httptest.NewRecorder()
handler.register(rec, req)

// Теперь можно проверять:
rec.Code          // HTTP-статус (int)
rec.Body.String() // тело ответа (string)
rec.Body.Bytes()  // тело как []byte
rec.Header()      // заголовки ответа
```

Обрати внимание: `httptest.NewRecorder()` — это **не** настоящий сервер. Обработчик вызывается напрямую, без сети.

## Уровень 6: про проверку JSON-ответа

```go
var resp map[string]any
if err := json.Unmarshal(rec.Body.Bytes(), &resp); err != nil {
    t.Fatalf("decode response: %v", err)
}

if id, ok := resp["id"].(float64); !ok || id == 0 {
    t.Errorf("id is zero or missing: %v", resp["id"])
}
```

⚠️ **Важная тонкость**: при `json.Unmarshal` в `map[string]any` числа парсятся как `float64`, **не** `int64`. Это особенность `encoding/json`. Чтобы получить `int64`, парси в конкретную структуру:

```go
var resp struct {
    ID int64 `json:"id"`
}
json.Unmarshal(rec.Body.Bytes(), &resp)
```

## Уровень 7: про testcontainers

Первый раз — удели время на установку и чтение документации. Минимум:

```go
import (
    "context"

    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    "time"
)

func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()

    ctx := context.Background()

    pgContainer, err := postgres.Run(ctx,
        "postgres:16",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        t.Fatalf("start container: %v", err)
    }

    t.Cleanup(func() {
        pgContainer.Terminate(ctx)
    })

    dsn, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        t.Fatalf("connection string: %v", err)
    }

    db, err := sql.Open("pgx", dsn)
    if err != nil {
        t.Fatalf("sql.Open: %v", err)
    }
    t.Cleanup(func() { db.Close() })

    // Применяем миграции
    if err := applyMigrations(db); err != nil {
        t.Fatalf("migrations: %v", err)
    }

    return db
}
```

## Уровень 8: про applyMigrations

Самый простой способ — прочитать все `.sql` файлы из папки `migrations/` и выполнить:

```go
import (
    "database/sql"
    "os"
    "path/filepath"
    "sort"
)

func applyMigrations(db *sql.DB) error {
    files, err := filepath.Glob("../../migrations/*.sql")
    if err != nil {
        return err
    }
    sort.Strings(files)  // чтобы применить по порядку 001, 002, 003

    for _, file := range files {
        data, err := os.ReadFile(file)
        if err != nil {
            return fmt.Errorf("read %s: %w", file, err)
        }
        if _, err := db.Exec(string(data)); err != nil {
            return fmt.Errorf("execute %s: %w", file, err)
        }
    }
    return nil
}
```

⚠️ **Путь `../../migrations/*.sql`** — потому что тесты запускаются из `internal/storage/`, а миграции лежат в корне проекта, на два уровня выше. Если у тебя другая структура — поправь путь.

Альтернатива — `embed.FS` из Go 1.16+:

```go
import "embed"

//go:embed migrations/*.sql
var migrationsFS embed.FS
```

Тогда миграции встраиваются в бинарник, и путь становится виртуальным. Это чище, но сложнее для модуля 6. Оставь для будущих проектов.

## Уровень 9: про graceful shutdown

Ключевые элементы:

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  15 * time.Second,
    WriteTimeout: 15 * time.Second,
    IdleTimeout:  60 * time.Second,
}

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("server failed: %v", err)
    }
}()

sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
<-sigChan

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Printf("shutdown error: %v", err)
}
```

Три критических момента:

1. **`srv.ListenAndServe()` в горутине** — иначе `main` застрянет здесь, и не дойдёт до signal.Notify.

2. **`err != http.ErrServerClosed`** — когда ты вызываешь `Shutdown`, `ListenAndServe` возвращает именно эту ошибку. Она **нормальная**. Её нужно игнорировать.

3. **`context.WithTimeout` на shutdown** — чтобы если активные запросы не завершатся за 10 секунд, сервер всё равно остановился.

## Уровень 10: про .env.example и README

Файл `.env.example` — это **пример** переменных без реальных значений:

```
DATABASE_URL=postgres://postgres:secret@localhost:5432/blog?sslmode=disable
JWT_SECRET=change-me-in-production
SERVER_ADDR=:8080
```

Настоящий `.env` — в `.gitignore`. Разработчик копирует `.env.example` в `.env` и правит значения.

Структура README для портфолио:

```markdown
# Blog Platform

Минимальный REST API блога: пользователи, посты, комментарии. Go + PostgreSQL + JWT.

## Stack
- Go 1.22+
- PostgreSQL 16
- `database/sql` + `jackc/pgx/v5` (без ORM)
- `golang-jwt/jwt/v5` (auth)
- `bcrypt` (пароли)
- `net/http` + path patterns (без фреймворков)
- `testcontainers` (integration тесты)

## Endpoints
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /register | — | Регистрация |
| POST | /login | — | Логин, получение JWT |
| GET | /me | ✓ | Текущий пользователь |
| GET | /posts | — | Список постов |
| GET | /posts/{id} | — | Один пост |
| POST | /posts | ✓ | Создать пост |
| PUT | /posts/{id} | ✓ (автор) | Обновить пост |
| DELETE | /posts/{id} | ✓ (автор) | Удалить пост |
| GET | /posts/{id}/comments | — | Комментарии поста |
| POST | /posts/{id}/comments | ✓ | Добавить комментарий |

## Quick start
\`\`\`bash
# 1. Postgres в Docker
docker run -d --name blog-pg -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:16

# 2. Миграции
psql -h localhost -U postgres -f migrations/001_create_users.sql
psql -h localhost -U postgres -f migrations/002_create_posts.sql
psql -h localhost -U postgres -f migrations/003_create_comments.sql

# 3. Сервер
cp .env.example .env
DATABASE_URL="..." JWT_SECRET="..." go run ./cmd/server
\`\`\`

## Tests
\`\`\`bash
go test ./...
\`\`\`
```

## Если совсем тупик

Скажи Claude конкретно: «модуль 6 проекта 7, [проблема]». Самые частые:

- **«testcontainers не запускается»** → проверь, что Docker работает (`docker ps`)
- **«тесты очень долгие»** → первый запуск качает образ `postgres:16`. Последующие — из кеша.
- **«applyMigrations не находит файлы»** → путь `../../migrations/*.sql` относительный, зависит от того, где запускается тест. Попробуй абсолютный или `embed.FS`.
- **«server failed: http: Server closed» на Ctrl+C** → забыл проверку `err != http.ErrServerClosed`
- **«Shutdown timeout»** → активные запросы не укладываются в 10 секунд. Либо увеличь, либо найди медленный запрос.
- **«context deadline exceeded» в тестах с testcontainers** → WaitStrategy не срабатывает. Проверь, что у `WithOccurrence(2)` именно 2 (для PostgreSQL это нужно).
