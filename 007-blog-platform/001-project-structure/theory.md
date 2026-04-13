# Модуль 1. Структура проекта — теория

## Контекст

В Проектах 1–6 ты держал весь код в **одном пакете `package main`**. Несколько файлов рядом, все видят друг друга — это работало для маленьких приложений. Но Проект 7 — другой:

- 6 модулей с десятками функций
- Несколько слоёв: HTTP-обработчики, бизнес-логика, работа с БД, аутентификация
- Сторонние зависимости (bcrypt, JWT)
- Тесты в отдельных файлах

В одном пакете это превратится в кашу из 30 файлов, где невозможно понять, что от чего зависит. Решение — **разделить код на пакеты**, каждый со своей ответственностью.

Этот модуль — про **архитектуру**. Ты не пишешь бизнес-логику. Ты создаёшь **скелет проекта**, в котором появятся следующие модули.

## Ядро 1: что такое пакет в реальном проекте

В Go **пакет = директория**. Все `.go` файлы в одной папке составляют один пакет. Имя пакета указывается в строке `package ...` в начале каждого файла.

Между пакетами действуют правила видимости через **регистр первой буквы**:

- **С большой буквы** (`User`, `GetByID`) — публичное, видно из других пакетов
- **С маленькой буквы** (`user`, `getByID`) — приватное, видно только внутри пакета

Это — **главный механизм инкапсуляции в Go**. Никаких ключевых слов `public`/`private` — только регистр.

В Проекте 6 ты уже использовал это правило, но в одном пакете оно не имело реального значения (`mu` с маленькой буквы спокойно использовался из любого файла модуля). Здесь — будет иметь.

## Ядро 2: стандартный layout Go-проектов

В Go-сообществе устоялся **квази-стандартный** layout, который ты увидишь в большинстве серьёзных проектов:

```
project/
├── go.mod
├── go.sum
├── cmd/
│   └── server/
│       └── main.go        ← точка входа: только инициализация
├── internal/
│   ├── api/               ← HTTP-обработчики, роутинг, middleware
│   ├── storage/           ← работа с БД (репозитории)
│   ├── auth/              ← JWT, bcrypt, аутентификация
│   ├── domain/            ← типы данных (User, Post, Comment)
│   └── config/            ← чтение env, валидация конфига
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_posts.sql
│   └── 003_create_comments.sql
├── README.md
└── .env.example           ← пример переменных окружения
```

Разберём по частям.

### cmd/server/main.go

Папка `cmd/` содержит **исполняемые программы**. Каждая подпапка внутри = отдельная программа. У нас одна — `cmd/server/main.go`, который запускает HTTP-сервер.

Если бы у проекта была ещё CLI-утилита для миграций — была бы `cmd/migrate/main.go`. И так далее.

`main.go` обычно очень **короткий** — он только:

1. Читает конфиг
2. Открывает БД
3. Создаёт зависимости (stores, services, handlers)
4. Регистрирует HTTP-роуты
5. Запускает сервер

Никакой бизнес-логики. Это **плотина** между внешним миром (env, БД, сеть) и твоим кодом.

### internal/

Папка `internal/` имеет **специальное значение** в Go: пакеты внутри неё **не могут быть импортированы** из других модулей. Это — встроенная защита от случайного «чужого» использования.

Например, если кто-то добавит твой проект как зависимость в свой:

```go
import "github.com/lev/blog/internal/auth"  // ❌ КОМПИЛЯТОР НЕ ДАСТ
```

Это полезно, потому что код в `internal/` — это **детали реализации**, не публичный API. Если ты хочешь, чтобы что-то было публичным API, кладёшь в корень или в `pkg/` (но `pkg/` сейчас используется реже).

**Все наши пакеты — в `internal/`.** У нас не библиотека, а приложение.

### internal/api

HTTP-слой: обработчики (`Handler`), регистрация роутов, middleware. Этот пакет:

- **Знает** про HTTP (`net/http`, `r.PathValue`, статусы)
- **Знает** про DTO (структуры запросов/ответов)
- **Не знает** про SQL (это `storage`)
- **Не знает** про bcrypt/JWT (это `auth`)

Его задача — принять HTTP-запрос, парсить, вызвать нужный сервис/репозиторий, превратить ответ в HTTP.

### internal/storage

Слой данных: репозитории, которые делают SQL-запросы. Этот пакет:

- **Знает** про `database/sql`, SQL-запросы, типы PostgreSQL
- **Знает** про domain-типы (`User`, `Post`)
- **Не знает** про HTTP
- **Не знает** про JWT, bcrypt

Это **изолированный** слой. Если завтра ты захочешь сменить PostgreSQL на MySQL — поправишь только этот пакет.

### internal/auth

Аутентификация: bcrypt для паролей, JWT для токенов. Этот пакет:

- **Знает** про криптографию (хеширование, подписи)
- **Знает** про JWT-формат
- **Не знает** про HTTP, БД, конкретные сущности

В нём будут функции вроде `HashPassword(plain string) string`, `CheckPassword(hash, plain string) bool`, `GenerateToken(userID int64) string`, `ValidateToken(token string) (int64, error)`.

### internal/domain

**Типы данных**, которые используются по всему проекту: `User`, `Post`, `Comment`. Их используют и `api`, и `storage`, и `auth`.

Это **общий словарь** проекта. Положить их в один из других пакетов было бы плохо: возникла бы зависимость (например, `api` зависит от `storage` только из-за типа `User`).

Альтернативно, можно держать domain-типы в `internal/storage/` (рядом с тем, кто их пишет в БД) и переоткрывать в `api`. Это менее чисто, но тоже встречается.

**Для нашего проекта — отдельный `internal/domain`.**

### internal/config

Чтение переменных окружения, валидация. Это маленький пакет, который часто опускают, но он полезен:

```go
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

Это **одна точка**, где читаются все env. Если завтра добавится новая переменная — добавляешь сюда, не разбрасываешь `os.Getenv` по всему коду.

## Ядро 3: модуль и module path

В корне проекта лежит `go.mod`:

```
module github.com/lev/blog

go 1.22

require (
    github.com/jackc/pgx/v5 v5.5.0
    github.com/golang-jwt/jwt/v5 v5.2.0
    golang.org/x/crypto v0.18.0
)
```

`module github.com/lev/blog` — это **module path**, корневой импорт. Все внутренние пакеты доступны через него:

```go
import "github.com/lev/blog/internal/storage"
import "github.com/lev/blog/internal/api"
```

**Module path не обязан быть реальным URL**, но **по соглашению** это путь к репозиторию (даже если репозитория ещё нет). Это удобно, когда проект становится публичным — `go get` работает по этому пути.

Для учебного проекта можешь использовать любое имя, например:
- `github.com/yourname/blog`
- `blog`  (без префикса — самый простой вариант)
- `lev/blog-platform`

## Ядро 4: dependency injection через конструкторы

В Go **нет** встроенного DI-фреймворка (как Spring в Java). Зависимости передаются **явно через конструкторы**:

```go
// В internal/storage/user.go
type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

// В internal/api/handlers.go
type Handler struct {
    users *storage.UserRepository
    auth  *auth.Service
}

func NewHandler(users *storage.UserRepository, auth *auth.Service) *Handler {
    return &Handler{users: users, auth: auth}
}

// В cmd/server/main.go
func main() {
    db := openDB(...)
    userRepo := storage.NewUserRepository(db)
    authService := auth.NewService("secret")
    handler := api.NewHandler(userRepo, authService)
    handler.RegisterRoutes(http.DefaultServeMux)
    http.ListenAndServe(":8080", nil)
}
```

`main` — это **точка**, где собираются все зависимости. Каждая структура принимает свои зависимости через конструктор. Это:

- **Тестируемо** — можно подменить `*sql.DB` на mock в тестах
- **Явно** — видно, что от чего зависит
- **Не магия** — нет автоматической генерации, нет рефлексии

В больших проектах используют DI-библиотеки (Wire, Fx) для генерации `main`-функции. Но для проекта на 5–10 зависимостей **ручной DI лучше**: проще читать, меньше магии.

## Ядро 5: что такое слои

«Слой» — это абстракция, которая помогает думать о коде.

Классическое разделение для веб-приложений:

```
┌─────────────────────────┐
│   HTTP / API layer      │  ← internal/api (handlers, middleware, DTO)
├─────────────────────────┤
│   Service / Use case    │  ← (для нас совмещено с handler в простых случаях)
├─────────────────────────┤
│   Domain                │  ← internal/domain (User, Post, Comment)
├─────────────────────────┤
│   Repository / Storage  │  ← internal/storage (SQL, *sql.DB)
└─────────────────────────┘
```

**Главное правило слоёв**: верхний слой **знает** про нижний, нижний **не знает** про верхний.

- `api` знает про `storage` (вызывает методы репозитория)
- `storage` **не знает** про `api` (никакого импорта `net/http` в репозитории!)
- Domain — общий, знают все

Если нарушаешь — получаешь **циклические зависимости**, которые Go запрещает на уровне компилятора. Это полезное ограничение: оно заставляет тебя думать про границы.

## Ядро 6: где жить «бизнес-логике»

В Проекте 7 у нас есть выбор:

### Вариант A: тонкий handler, толстый repository

```go
// api/post.go
func (h *Handler) createPost(w, r) {
    // парсинг ввода
    // валидация
    post, err := h.posts.Create(...)  // вся логика здесь
    // ответ
}

// storage/post.go
func (r *PostRepository) Create(...) (*Post, error) {
    // INSERT INTO posts ...
    // возможно, с дополнительной логикой
}
```

### Вариант B: тонкий handler, тонкий repository, отдельный service

```go
// api/post.go
func (h *Handler) createPost(w, r) {
    // парсинг
    // валидация
    post, err := h.postService.Create(...)
}

// service/post.go
type PostService struct {
    repo *storage.PostRepository
}
func (s *PostService) Create(...) (*Post, error) {
    // бизнес-логика
    // вызов репозитория
}

// storage/post.go
func (r *PostRepository) Insert(...) (*Post, error) {
    // только SQL
}
```

**Вариант B чище для больших проектов**, но требует больше слоёв. **Вариант A проще для нашего**.

Для проекта 7 мы используем **Вариант A**: handler парсит, валидирует, вызывает repository. Сложной бизнес-логики у нас нет, отдельный service-слой стал бы лишним.

В реальном большом проекте (десятки эндпоинтов, сложная логика) переходишь на Вариант B. Это **эволюция архитектуры** — она зависит от размера, не от моды.

## Ядро 7: финальная структура для модуля 1

Что у тебя должно появиться к концу модуля:

```
007-blog-platform/
├── go.mod
├── go.sum
├── README.md
├── .env.example
├── cmd/
│   └── server/
│       └── main.go        ← inicialización + вывод "Hello, blog"
├── internal/
│   ├── api/
│   │   └── handler.go     ← заглушка Handler с одним методом hello
│   ├── auth/
│   │   └── service.go     ← заглушка Service
│   ├── config/
│   │   └── config.go      ← Load() с DATABASE_URL и JWT_SECRET
│   ├── domain/
│   │   └── user.go        ← заглушка типа User
│   └── storage/
│       └── user.go        ← заглушка UserRepository
└── migrations/
    └── (пусто — наполним в модуле 2)
```

И `main.go`:

```go
package main

import (
    "log"
    "net/http"

    "github.com/yourname/blog/internal/api"
    "github.com/yourname/blog/internal/auth"
    "github.com/yourname/blog/internal/config"
    "github.com/yourname/blog/internal/storage"
)

func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatalf("config: %v", err)
    }

    // Заглушки для модуля 1 — без БД
    userRepo := storage.NewUserRepository(nil)
    authService := auth.NewService(cfg.JWTSecret)
    handler := api.NewHandler(userRepo, authService)

    handler.RegisterRoutes(http.DefaultServeMux)

    log.Printf("Сервер запускается на :%s", cfg.Port)
    if err := http.ListenAndServe(":"+cfg.Port, nil); err != nil {
        log.Fatalf("server: %v", err)
    }
}
```

И один эндпоинт `GET /hello`, который проверяет, что цепочка работает:

```go
// internal/api/handler.go
package api

import (
    "fmt"
    "net/http"

    "github.com/yourname/blog/internal/auth"
    "github.com/yourname/blog/internal/storage"
)

type Handler struct {
    users *storage.UserRepository
    auth  *auth.Service
}

func NewHandler(users *storage.UserRepository, auth *auth.Service) *Handler {
    return &Handler{users: users, auth: auth}
}

func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    mux.HandleFunc("GET /hello", h.hello)
}

func (h *Handler) hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, blog!")
}
```

Запуск:

```bash
DATABASE_URL=postgres://... JWT_SECRET=test go run ./cmd/server
```

И:

```bash
$ curl http://localhost:8080/hello
Hello, blog!
```

Это «hello world» структурированного проекта. Кода в нём почти нет, но **скелет** уже стоит.

## Ядро 8: команды для запуска

В отличие от предыдущих проектов, теперь main лежит в `cmd/server/main.go`, а не в корне. Команды:

```bash
# Запуск
go run ./cmd/server

# Сборка
go build -o blog ./cmd/server
./blog

# Запуск тестов
go test ./...
```

`./...` означает «все пакеты во всех подпапках». Полезно для тестов и форматирования.

## Вопросы на понимание

1. Что такое **пакет** в Go и какие у него правила видимости?
2. Что особенного в папке `internal/`?
3. Какая разница между `cmd/` и `internal/`?
4. Что должно быть в `main.go` стандартизированного проекта?
5. Что такое **dependency injection** через конструкторы? Чем оно лучше глобальных переменных?
6. Какое **главное правило слоёв** в архитектуре приложения?
7. Что делает Go, если есть **циклическая зависимость** между пакетами?
8. Зачем нужен пакет `internal/config`?
9. Чем `Вариант A` (тонкий handler, толстый repository) отличается от `Вариант B` (отдельный service)? Какой выбран и почему?
10. Что означает `go run ./cmd/server`?
