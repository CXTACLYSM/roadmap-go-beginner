# Модуль 1. Структура проекта — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какая папка имеет специальное значение в Go?
2. Где должен быть `main.go` в стандартном layout?
3. Что нужно указать первой строкой `go.mod`?
4. Как импортировать пакет внутри своего модуля?
5. Какие имена пакетов соответствуют папкам?

## Уровень 2: про go mod init

```bash
go mod init blog
```

Это создаёт `go.mod` с одной строкой:

```
module blog

go 1.22
```

`blog` — это **module path**. Все импорты внутренних пакетов будут начинаться с этого:

```go
import "blog/internal/api"
import "blog/internal/storage"
```

Если ты выбрал `github.com/yourname/blog`:

```go
import "github.com/yourname/blog/internal/api"
```

Длиннее, но более «правильно» с точки зрения публикации.

**Используй короткое имя `blog` для учебного проекта.** Меняется одной командой `go mod edit -module github.com/...`, если потом захочешь.

## Уровень 3: про package для каждого файла

Имя пакета должно совпадать с именем папки (не обязательно, но **по соглашению**):

- `internal/api/handler.go` → `package api`
- `internal/storage/user.go` → `package storage`
- `internal/auth/service.go` → `package auth`

Имя пакета **в первой строке файла**:

```go
package api

import (
    "net/http"
)

// ... код пакета api ...
```

Все файлы одной папки должны иметь **одно и то же** имя пакета.

## Уровень 4: про конфликты типов

В разных пакетах могут быть типы с одинаковыми именами. При импорте они доступны через **префикс пакета**:

```go
// internal/storage/user.go
package storage

type User struct { ... }

// cmd/server/main.go
import (
    "blog/internal/domain"
    "blog/internal/storage"
)

func main() {
    var u1 domain.User
    var u2 storage.User  // другой User
}
```

В нашем проекте у нас один `User` — в `internal/domain`. `storage` его **импортирует**:

```go
// internal/storage/user.go
package storage

import "blog/internal/domain"

type UserRepository struct {
    db *sql.DB
}

func (r *UserRepository) GetByID(id int64) (*domain.User, error) { ... }
```

## Уровень 5: про config.Load()

```go
package config

import (
    "errors"
    "os"
)

type Config struct {
    DatabaseURL string
    JWTSecret   string
    Port        string
}

func Load() (*Config, error) {
    cfg := &Config{
        DatabaseURL: os.Getenv("DATABASE_URL"),
        JWTSecret:   os.Getenv("JWT_SECRET"),
        Port:        os.Getenv("PORT"),
    }

    if cfg.DatabaseURL == "" {
        return nil, errors.New("DATABASE_URL is required")
    }
    if cfg.JWTSecret == "" {
        return nil, errors.New("JWT_SECRET is required")
    }
    if cfg.Port == "" {
        cfg.Port = "8080"
    }

    return cfg, nil
}
```

Несколько моментов:

- **Возвращаем `*Config`**, а не `Config` — указатель удобнее передавать
- **`errors.New(...)`** для простых статических ошибок
- **Дефолт для `Port`** — если пусто, ставим 8080

## Уровень 6: про Handler с полями-заглушками

```go
// internal/api/handler.go
package api

import (
    "fmt"
    "net/http"

    "blog/internal/auth"
    "blog/internal/storage"
)

type Handler struct {
    users *storage.UserRepository
    auth  *auth.Service
}

func NewHandler(users *storage.UserRepository, auth *auth.Service) *Handler {
    return &Handler{
        users: users,
        auth:  auth,
    }
}

func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    mux.HandleFunc("GET /hello", h.hello)
}

func (h *Handler) hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, blog!")
}
```

Заметь:

- **Поля приватные** (`users`, `auth`) — наружу не нужны
- **Конструктор `NewHandler` публичный** — он создаётся снаружи (из `main`)
- **Имена полей и параметров одинаковые** — это не запрещено, разрешено и идиоматично

## Уровень 7: про main.go

```go
// cmd/server/main.go
package main

import (
    "log"
    "net/http"

    "blog/internal/api"
    "blog/internal/auth"
    "blog/internal/config"
    "blog/internal/storage"
)

func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatalf("config: %v", err)
    }

    // Заглушки: nil вместо БД, потому что в модуле 1 БД ещё не нужна
    userRepo := storage.NewUserRepository(nil)
    authService := auth.NewService(cfg.JWTSecret)
    handler := api.NewHandler(userRepo, authService)

    mux := http.NewServeMux()
    handler.RegisterRoutes(mux)

    addr := ":" + cfg.Port
    log.Printf("Сервер запускается на %s", addr)
    if err := http.ListenAndServe(addr, mux); err != nil {
        log.Fatalf("server: %v", err)
    }
}
```

Обрати внимание:

1. **`http.NewServeMux()`** вместо `http.DefaultServeMux` — это **свой** мультиплексор, который мы передаём в `ListenAndServe`. Чище, чем глобальный.

2. **Передача `mux` в `ListenAndServe`** вместо `nil`. С `nil` использовался бы default, теперь — наш собственный.

3. **`handler.RegisterRoutes(mux)`** — handler знает, как зарегистрировать свои роуты в любом mux. Это удобно для тестирования (можно передать тестовый mux).

## Уровень 8: про nil вместо *sql.DB

В модуле 1 у нас ещё нет БД, поэтому передаём `nil`:

```go
userRepo := storage.NewUserRepository(nil)
```

Это **временное** решение. В модуле 2 заменим на реальный `*sql.DB`. **Но** методы репозитория, вызывающие `db.Query(...)` на nil, упадут с panic. Чтобы не упасть в модуле 1, репозиторий не должен иметь методов, использующих `db`. Только заглушки или комментарии.

## Уровень 9: про .env.example

```
DATABASE_URL=postgres://postgres:secret@localhost:5432/blog?sslmode=disable
JWT_SECRET=change-me-in-production
PORT=8080
```

Это **пример** — список переменных, которые ожидает приложение. Его коммитят в git. Реальные значения (`.env`) — нет (в `.gitignore`).

Это стандартный приём, чтобы новый разработчик мог открыть `.env.example`, скопировать в `.env`, заполнить — и запустить.

## Уровень 10: про запуск с переменными окружения

В Linux/macOS:

```bash
DATABASE_URL=test JWT_SECRET=test go run ./cmd/server
```

В Windows (cmd):

```cmd
set DATABASE_URL=test
set JWT_SECRET=test
go run ./cmd/server
```

В Windows (PowerShell):

```powershell
$env:DATABASE_URL="test"
$env:JWT_SECRET="test"
go run ./cmd/server
```

Альтернативно — экспортировать на сессию:

```bash
export DATABASE_URL="test"
export JWT_SECRET="test"
go run ./cmd/server
```

После `export` переменные доступны до конца сессии терминала.

## Если совсем тупик

Скажи Claude конкретно: «модуль 1 проекта 7, [конкретная проблема]». Самые частые ситуации:

- **«package not found»** → проверь, что module path в `go.mod` совпадает с импортами
- **«import cycle not allowed»** → ищи, какие два пакета друг друга импортируют
- **«undefined: ..."** → не экспортирована функция (с маленькой буквы) или забыл импорт
- **«no main function»** → файл не в `package main` или нет `func main()`
- **«unknown command "run ./cmd/server"»** → ты не в корне модуля (нет `go.mod` в текущей папке)
