# Модуль 6. Тесты и финализация — теория

## Контекст

Это **финальный модуль** Проекта 7 и всего roadmap-а. Твой блог функционально работает: пользователи регистрируются, логинятся, пишут посты, комментируют. Осталось превратить это из «работающей программы» в **надёжный сервис**.

В этом модуле ты:

- Напишешь **unit-тесты** для `auth.Service` (без БД)
- Напишешь **тесты обработчиков** через `httptest.NewRecorder` (без реальной сети)
- Напишешь **integration-тесты** для `UserRepository` через **testcontainers** (с настоящей PostgreSQL в Docker)
- Добавишь **graceful shutdown** через `http.Server.Shutdown`
- Оформишь проект для **портфолио**: README, пример .env, таблица эндпоинтов

После этого модуля у тебя будет проект, который не стыдно показать на собеседовании.

## Ядро 1: зачем вообще тесты

Тесты — это не про «правильный подход» и не про моду. Это про **уверенность при изменении кода**. Без тестов:

- Ты боишься трогать работающий код
- Рефакторинг = лотерея («вроде всё ещё работает… наверное»)
- Баги возвращаются после каждого релиза
- Onboarding нового разработчика = неделя медленной боли

С тестами:

- Ты меняешь код смело: если тесты зелёные, ничего не сломалось
- CI ловит баги до того, как они попадут в продакшен
- Тесты **сами** документируют, как должен себя вести код

Но тесты — это **инструмент**, а не самоцель. Цель — **покрыть важные пути** так, чтобы большинство багов ловились автоматически. Покрытие 100% — недостижимый и вредный фетиш. Покрытие 0% — халатность.

В этом модуле ты покроешь **три уровня** твоего блога:

1. **Чистую логику** без зависимостей (`auth.Service` — хеширование, JWT)
2. **HTTP-обработчики** с моками (через `httptest`)
3. **БД-операции** с реальной PostgreSQL (через `testcontainers`)

Этого достаточно для **портфолио-проекта**. В продакшен-системах добавляют ещё end-to-end тесты, load-тесты, chaos-тесты — но это уже отдельные дисциплины.

## Ядро 2: `testing` пакет в Go

В Go тесты — это **часть стандартной библиотеки**. Никаких RSpec, Jest, PyTest — только пакет `testing`.

Файлы тестов называются `*_test.go`. Функции тестов — `func TestXxx(t *testing.T)`, где `Xxx` — осмысленное имя.

Минимальный тест:

```go
// internal/auth/service_test.go
package auth

import "testing"

func TestHashPassword(t *testing.T) {
    svc := NewService("secret")
    hash, err := svc.HashPassword("password123")
    if err != nil {
        t.Fatalf("HashPassword: %v", err)
    }
    if hash == "password123" {
        t.Errorf("хеш совпадает с паролем — bcrypt не отработал")
    }
}
```

Несколько ключевых моментов:

1. **`func TestXxx(t *testing.T)`** — обязательная сигнатура
2. **`t.Errorf(...)`** — помечает тест как упавший, но **продолжает** выполнение
3. **`t.Fatalf(...)`** — помечает как упавший и **останавливает** этот конкретный тест
4. **`t.Logf(...)`** — пишет в лог (видно только при `-v` или если тест упал)

Когда использовать что:

- **`Fatalf`** — если дальше тест не имеет смысла (например, объект не создался, продолжать нечего)
- **`Errorf`** — если продолжение осмысленно (можно проверить ещё несколько условий и показать все упавшие)

Запуск:

```bash
go test ./...           # все тесты
go test ./internal/auth # конкретный пакет
go test -v ./...        # с подробным выводом
go test -run TestHash   # только тесты с TestHash в имени
```

## Ядро 3: table-driven tests

Самый идиоматичный паттерн Go-тестов. Вместо дублирования кода «проверить одно, проверить другое, проверить третье» — таблица случаев и один цикл.

```go
func TestGenerateToken(t *testing.T) {
    svc := NewService("test-secret")

    cases := []struct {
        name   string
        userID int64
        wantOK bool
    }{
        {"valid user id", 1, true},
        {"zero user id", 0, true},        // технически валидно
        {"negative user id", -1, true},   // тоже валидно
    }

    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            token, err := svc.GenerateToken(tc.userID)
            if tc.wantOK && err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if !tc.wantOK && err == nil {
                t.Fatalf("expected error, got nil")
            }
            if tc.wantOK && token == "" {
                t.Errorf("token is empty")
            }
        })
    }
}
```

Несколько важных моментов:

1. **`cases := []struct{...}{...}`** — слайс анонимных структур. Каждый элемент — один тестовый случай с входом и ожидаемым результатом.

2. **`t.Run(name, func(t *testing.T) {...})`** — подтест. Каждый случай получает **своё** имя в выводе, и при падении видно, какой именно упал:
   ```
   --- FAIL: TestGenerateToken/valid_user_id
   ```

3. **Переменная `tc` — копия элемента слайса**. В Go < 1.22 была ловушка: горутина внутри теста захватывала `tc` из цикла и видела всегда последний элемент. С Go 1.22+ это исправлено (`range` создаёт новую переменную каждой итерации).

4. **`name` в структуре** — это обязательное поле. Без него не получишь хорошие имена подтестов.

## Ядро 4: unit-тесты для auth.Service

Начнём с самого простого — **чистой логики без внешних зависимостей**. `auth.Service` умеет хешировать пароли и генерировать/проверять JWT. БД не нужна, сеть не нужна.

```go
// internal/auth/service_test.go
package auth

import (
    "testing"
)

func TestService_HashAndCheck(t *testing.T) {
    svc := NewService("test-secret")

    password := "correcthorsebatterystaple"

    hash, err := svc.HashPassword(password)
    if err != nil {
        t.Fatalf("HashPassword: %v", err)
    }

    // Хеш не совпадает с паролем
    if hash == password {
        t.Errorf("hash equals password")
    }

    // Правильный пароль проходит
    if err := svc.CheckPassword(hash, password); err != nil {
        t.Errorf("CheckPassword with correct password: %v", err)
    }

    // Неправильный — нет
    if err := svc.CheckPassword(hash, "wrong"); err == nil {
        t.Errorf("CheckPassword with wrong password: expected error, got nil")
    }
}

func TestService_GenerateAndValidateToken(t *testing.T) {
    svc := NewService("test-secret")

    token, err := svc.GenerateToken(42)
    if err != nil {
        t.Fatalf("GenerateToken: %v", err)
    }

    userID, err := svc.ValidateToken(token)
    if err != nil {
        t.Fatalf("ValidateToken: %v", err)
    }

    if userID != 42 {
        t.Errorf("user id = %d, want 42", userID)
    }
}

func TestService_ValidateToken_WrongSecret(t *testing.T) {
    svc1 := NewService("secret-one")
    svc2 := NewService("secret-two")

    token, _ := svc1.GenerateToken(42)

    _, err := svc2.ValidateToken(token)
    if err == nil {
        t.Error("expected error for token with wrong secret, got nil")
    }
}
```

Эти тесты работают **мгновенно** — миллисекунды. Никаких зависимостей. Ты можешь запускать их в любом IDE, на любом компьютере, в любой момент.

**Правило:** чистая логика должна быть покрыта такими тестами в первую очередь. Они самые дешёвые и самые полезные.

## Ядро 5: httptest для HTTP-обработчиков

Для тестирования обработчиков есть специальный пакет `net/http/httptest`. Он даёт:

- **`httptest.NewRecorder()`** — фейковый `ResponseWriter`, куда обработчик пишет ответ
- **`httptest.NewRequest(method, url, body)`** — фейковый запрос, без сетевого слоя
- **`httptest.NewServer(handler)`** — реальный HTTP-сервер на случайном порту (для E2E)

Самый частый паттерн — **`NewRecorder` + `NewRequest`**. Они работают **без сети**: всё происходит в памяти, обработчик вызывается напрямую.

```go
// internal/api/auth_test.go
package api

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestHandler_Register_InvalidJSON(t *testing.T) {
    h := newTestHandler(t)  // helper

    req := httptest.NewRequest(
        http.MethodPost,
        "/register",
        bytes.NewReader([]byte("не json")),
    )
    req.Header.Set("Content-Type", "application/json")

    rec := httptest.NewRecorder()
    h.register(rec, req)

    if rec.Code != http.StatusBadRequest {
        t.Errorf("code = %d, want %d", rec.Code, http.StatusBadRequest)
    }
}
```

Разберём:

1. **`httptest.NewRequest(method, path, body)`** создаёт `*http.Request` без настоящей сети. `body` — это `io.Reader`, для JSON удобно `bytes.NewReader([]byte("..."))`
2. **`httptest.NewRecorder()`** создаёт `*httptest.ResponseRecorder`, который реализует `http.ResponseWriter` и **записывает** всё, что в него написали
3. **`h.register(rec, req)`** — вызываем handler напрямую. Никакого HTTP-сервера.
4. **`rec.Code`** — статус ответа. **`rec.Body`** — тело как `*bytes.Buffer`. **`rec.Header()`** — заголовки ответа.

Для JSON-ответа:

```go
var got map[string]any
if err := json.Unmarshal(rec.Body.Bytes(), &got); err != nil {
    t.Fatalf("decode response: %v", err)
}
if got["email"] != "lev@example.com" {
    t.Errorf("email = %v, want lev@example.com", got["email"])
}
```

## Ядро 6: проблема с зависимостями в handler-тестах

Handler зависит от `UserRepository` и `auth.Service`. Для unit-теста нужно **не тянуть реальную БД**. Есть три варианта:

### Вариант A: интерфейс + мок

```go
// internal/api/handler.go
type UserRepo interface {
    Create(ctx context.Context, email, hash string) (*domain.User, error)
    GetByEmail(ctx context.Context, email string) (*domain.User, error)
}

type Handler struct {
    users UserRepo  // теперь интерфейс, а не конкретный тип
    auth  *auth.Service
}
```

И в тесте:

```go
type mockUserRepo struct {
    createFn func(ctx context.Context, email, hash string) (*domain.User, error)
}

func (m *mockUserRepo) Create(ctx context.Context, email, hash string) (*domain.User, error) {
    return m.createFn(ctx, email, hash)
}
// ... аналогично GetByEmail ...
```

**Плюсы**: полный контроль, можно симулировать ошибки БД.
**Минусы**: нужно определять интерфейсы, писать моки, поддерживать их при изменениях.

### Вариант B: integration-тесты с testcontainers

Не пишем моки, а запускаем реальную PostgreSQL в Docker на время теста. Код не меняется, тесты работают с настоящей БД.

**Плюсы**: тестируется реальный SQL, ловим баги, которые моки не ловят.
**Минусы**: медленнее (секунды вместо миллисекунд), нужен Docker.

### Вариант C: гибрид

- **Unit-тесты** для `auth.Service` (чистая логика)
- **Integration-тесты** для `UserRepository` через testcontainers
- **Handler-тесты** — либо оба (unit с моками + integration через `httptest.NewServer`), либо только integration через реальную БД

**Для нашего проекта — Вариант C.** Это даёт хорошее покрытие без чрезмерной сложности.

## Ядро 7: testcontainers для integration-тестов

**testcontainers-go** — библиотека, которая запускает Docker-контейнеры из тестов, подключается к ним и автоматически убирает после.

Установка:

```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

Использование:

```go
// internal/storage/user_test.go
package storage

import (
    "context"
    "database/sql"
    "testing"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
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
        t.Fatalf("start postgres: %v", err)
    }

    t.Cleanup(func() {
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("terminate container: %v", err)
        }
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

    if err := applyMigrations(db); err != nil {
        t.Fatalf("migrations: %v", err)
    }

    return db
}

func TestUserRepository_Create(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)

    user, err := repo.Create(context.Background(), "test@example.com", "hashed_password")
    if err != nil {
        t.Fatalf("Create: %v", err)
    }

    if user.ID == 0 {
        t.Errorf("user id is zero")
    }
    if user.Email != "test@example.com" {
        t.Errorf("email = %q, want %q", user.Email, "test@example.com")
    }
}

func TestUserRepository_Create_Duplicate(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)
    ctx := context.Background()

    _, err := repo.Create(ctx, "dup@example.com", "hash1")
    if err != nil {
        t.Fatalf("first create: %v", err)
    }

    _, err = repo.Create(ctx, "dup@example.com", "hash2")
    if !errors.Is(err, ErrEmailExists) {
        t.Errorf("expected ErrEmailExists, got %v", err)
    }
}
```

Ключевые моменты:

1. **`t.Helper()`** — помечает функцию как helper. При падении теста `t.Errorf` покажет не строку в helper-е, а строку **вызова** из теста. Важно для читаемости.

2. **`postgres.Run(ctx, "postgres:16", ...)`** — запускает контейнер. Возвращает объект, через который можно получить DSN.

3. **`wait.ForLog(...)`** — testcontainers ждёт, пока в логе появится нужная строка. Для PostgreSQL это «database system is ready to accept connections» — причём **два раза** (`WithOccurrence(2)`), потому что первый раз — инициализация, второй — реальная готовность.

4. **`t.Cleanup(fn)`** — функция вызывается при завершении теста (успехе или падении). Как `defer`, но для testing.

5. **`applyMigrations(db)`** — твой helper, который применяет `.sql` файлы из `migrations/`. Реализация зависит от тебя (читай файлы, выполняй через `db.Exec`).

Каждый тест получает **свой** контейнер, свою БД, свои миграции. Между тестами нет взаимного влияния. Это дорого по времени (несколько секунд на тест), но **абсолютно надёжно**.

**Альтернатива** — один контейнер на весь пакет через `TestMain`. Быстрее, но требует ручной очистки данных между тестами. Для нашего проекта начни с одного-контейнера-на-тест, при росте переходи на общий.

## Ядро 8: http.Server и graceful shutdown

В Проектах 4-5 ты использовал `http.ListenAndServe(":8080", nil)` — это коротко, но не даёт управлять жизненным циклом сервера. Для graceful shutdown нужен **явный `http.Server`**:

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  15 * time.Second,
    WriteTimeout: 15 * time.Second,
    IdleTimeout:  60 * time.Second,
}

// Запускаем в горутине, чтобы main мог слушать сигналы
go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("server failed: %v", err)
    }
}()

// Ждём сигнал
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
<-sigChan
log.Println("shutdown signal received")

// Graceful shutdown с таймаутом
shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Printf("shutdown error: %v", err)
}

log.Println("server stopped")
```

Что происходит при `Shutdown`:

1. **Закрывается listener** — новые запросы больше не принимаются
2. **Ждут активные запросы** — те, что уже начали обрабатываться, **доделываются**
3. **Если за `shutdownCtx` deadline не уложились** — возвращается ошибка, принудительное завершение

Важные детали:

- **`http.ErrServerClosed`** — специальная ошибка, которую возвращает `ListenAndServe` при нормальном shutdown. Её **нужно игнорировать** (не падать). Любая **другая** ошибка — это реальная проблема, нужно `log.Fatal`.

- **Таймауты `ReadTimeout` / `WriteTimeout`** — защита от медленных клиентов. Без них атакующий может открыть соединение и ничего не слать — сервер будет держать его вечно. Это **атака Slowloris**.

- **`IdleTimeout`** — сколько держать keep-alive соединение без запросов.

- **`defer db.Close()`** — после shutdown закрываем БД. Это гарантирует, что все транзакции завершены.

## Ядро 9: финальный main.go

Всё вместе:

```go
// cmd/server/main.go
package main

import (
    "context"
    "database/sql"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"

    "blog/internal/api"
    "blog/internal/auth"
    "blog/internal/config"
    "blog/internal/storage"
)

func main() {
    cfg := config.Load()

    db, err := sql.Open("pgx", cfg.DatabaseURL)
    if err != nil {
        log.Fatalf("sql.Open: %v", err)
    }
    defer db.Close()

    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)

    if err := db.Ping(); err != nil {
        log.Fatalf("db.Ping: %v", err)
    }

    userRepo := storage.NewUserRepository(db)
    postRepo := storage.NewPostRepository(db)
    commentRepo := storage.NewCommentRepository(db)
    authSvc := auth.NewService(cfg.JWTSecret)

    handler := api.NewHandler(userRepo, postRepo, commentRepo, authSvc)

    mux := http.NewServeMux()
    handler.RegisterRoutes(mux)

    srv := &http.Server{
        Addr:         cfg.Addr,
        Handler:      mux,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    go func() {
        log.Printf("server starting on %s", cfg.Addr)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server failed: %v", err)
        }
    }()

    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    sig := <-sigChan
    log.Printf("received signal %v, shutting down", sig)

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Printf("shutdown error: %v", err)
    }

    log.Println("server stopped")
}
```

Это — **production-грамотная** точка входа. Сравни с `http.ListenAndServe(":8080", nil)` из Проекта 4 — видно, насколько выросло понимание.

## Ядро 10: test pyramid

Общая рекомендация по балансу тестов — **пирамида**:

```
         /\
        /  \      E2E (мало, медленные, дорогие)
       /____\
      /      \    Integration (средне, с реальными зависимостями)
     /________\
    /          \  Unit (много, быстрые, дешёвые)
   /____________\
```

- **Unit-тесты** — много, быстрые (миллисекунды), без зависимостей. Для нас — `auth.Service`.
- **Integration-тесты** — меньше, медленнее (секунды), с реальными зависимостями. Для нас — `UserRepository` через testcontainers.
- **E2E-тесты** — ещё меньше, ещё медленнее (минуты), проверяют всю систему как черный ящик. Для нас — не делаем, но можно добавить через `httptest.NewServer`.

**Правило**: чем выше в пирамиде, тем дороже поддержка. Каждый уровень должен оправдывать свою стоимость.

Для твоего портфолио-проекта хватит **unit + integration**. E2E — это отдельная тема для будущих проектов.

## Вопросы на понимание

1. Какой пакет стандартной библиотеки Go отвечает за тесты?
2. Как называются файлы тестов? Какую сигнатуру должны иметь тестовые функции?
3. В чём разница между `t.Errorf` и `t.Fatalf`?
4. Что такое **table-driven test** и зачем он нужен?
5. Что делает `httptest.NewRecorder` и как его использовать?
6. Что делает `testcontainers-go` и в каких случаях он предпочтительнее моков?
7. Что такое `t.Helper()` и `t.Cleanup()`?
8. Что возвращает `ListenAndServe` при нормальном `Shutdown`? Как правильно её обработать?
9. Зачем нужны `ReadTimeout` и `WriteTimeout` на `http.Server`?
10. Что такое **test pyramid** и какие уровни она описывает?
