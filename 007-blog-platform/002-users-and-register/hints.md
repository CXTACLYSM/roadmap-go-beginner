# Модуль 2. Users и регистрация — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какая библиотека Go используется для bcrypt?
2. Какие два метода bcrypt тебе нужны?
3. Какой код ошибки PostgreSQL означает «дубликат UNIQUE»?
4. Какой HTTP-статус возвращать при «email уже существует»?
5. Какой JSON-тег скрывает поле от сериализации?

## Уровень 2: про bcrypt API

```go
import "golang.org/x/crypto/bcrypt"

// Хеширование
hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
if err != nil {
    return "", err
}
// hash — это []byte. Конвертируй в string для БД:
hashStr := string(hash)

// Проверка
err := bcrypt.CompareHashAndPassword([]byte(hashStr), []byte(password))
if err == nil {
    // совпало
}
if errors.Is(err, bcrypt.ErrMismatchedHashAndPassword) {
    // не совпало
}
```

Заметь:

- **`[]byte`** для пароля и хеша — bcrypt работает с байтами
- **`string(hash)`** — простая конвертация без копирования
- **`bcrypt.DefaultCost = 10`** — стандарт. Можно поставить выше для большей безопасности

## Уровень 3: про json:"-"

```go
type User struct {
    ID           int64     `json:"id"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`            // ← не сериализуется
    CreatedAt    time.Time `json:"created_at"`
}
```

Тег `json:"-"` (дефис) — это специальное значение. `encoding/json` **полностью игнорирует** поле. Оно не попадает в JSON ни при `Marshal`, ни при `Unmarshal`.

Это важная **страховка**. Даже если кто-то по ошибке вернёт `&user` целиком — пароль не утечёт.

## Уровень 4: про unique violation в PostgreSQL

Когда ты пытаешься вставить дубликат UNIQUE, PostgreSQL возвращает ошибку с **кодом** `23505`. Через pgx-драйвер это:

```go
import (
    "errors"
    "github.com/jackc/pgx/v5/pgconn"
)

func isUniqueViolation(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr.Code == "23505"
    }
    return false
}
```

Разберём:

- **`errors.As(err, &pgErr)`** — «есть ли в цепочке ошибок (включая обёрнутые) ошибка типа `*pgconn.PgError`?» Если да — записывает в `pgErr`.
- **`pgErr.Code`** — строка-код ошибки PostgreSQL. Список кодов в [документации PostgreSQL](https://www.postgresql.org/docs/current/errcodes-appendix.html).
- **`23505`** — `unique_violation`.

Использование:

```go
if isUniqueViolation(err) {
    return nil, ErrEmailExists
}
```

## Уровень 5: про sentinel errors

```go
// internal/storage/errors.go
package storage

import "errors"

var (
    ErrUserNotFound = errors.New("user not found")
    ErrEmailExists  = errors.New("email already exists")
)
```

Это **переменные пакета**. Объявляются один раз, используются везде. Внешний код проверяет через `errors.Is`:

```go
user, err := repo.Create(...)
if err != nil {
    if errors.Is(err, storage.ErrEmailExists) {
        // обработка
    }
    // другая ошибка
}
```

Преимущества:
- Чёткие категории ошибок
- Не нужно парсить сообщения
- Легко тестировать

## Уровень 6: про шаблон Create

```go
func (r *UserRepository) Create(ctx context.Context, email, passwordHash string) (*domain.User, error) {
    var u domain.User
    err := r.db.QueryRowContext(ctx, `
        INSERT INTO users (email, password_hash)
        VALUES ($1, $2)
        RETURNING id, email, password_hash, created_at
    `, email, passwordHash).Scan(&u.ID, &u.Email, &u.PasswordHash, &u.CreatedAt)

    if err != nil {
        if isUniqueViolation(err) {
            return nil, ErrEmailExists
        }
        return nil, fmt.Errorf("create user: %w", err)
    }

    return &u, nil
}
```

Структура:

1. `INSERT ... RETURNING ...` — одним запросом
2. Scan в поля `&u.ID, &u.Email, ...` (порядок совпадает с RETURNING)
3. Проверка ошибки → если unique violation → доменная ошибка
4. Иначе оборачиваем через `%w`
5. Возврат указателя на пользователя

## Уровень 7: про регулярку для email

```go
import "regexp"

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

if !emailRegex.MatchString(input.Email) {
    writeError(w, http.StatusBadRequest, "invalid email")
    return
}
```

`regexp.MustCompile` — это парсинг регулярки **при инициализации**. Если регулярка невалидна, программа упадёт на старте (а не при первом использовании). Это и есть смысл `Must` в имени.

`emailRegex.MatchString(s)` — проверка, соответствует ли строка регулярке.

Для production-системы лучше использовать `net/mail`:

```go
import "net/mail"

_, err := mail.ParseAddress(input.Email)
if err != nil {
    // невалидный email
}
```

Это парсер по RFC 5322. Покрывает все edge cases. Для нашего проекта — регулярка достаточна, но **знать про `net/mail`** обязательно.

## Уровень 8: про helpers writeJSON и writeError

```go
// internal/api/response.go
package api

import (
    "encoding/json"
    "log"
    "net/http"
)

func writeJSON(w http.ResponseWriter, status int, value any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(value); err != nil {
        log.Printf("write json: %v", err)
    }
}

func writeError(w http.ResponseWriter, status int, message string) {
    writeJSON(w, status, map[string]string{"error": message})
}
```

Это **не методы** на `Handler`, а просто функции в пакете `api`. Они доступны всем handler-ам пакета.

Использование:
```go
writeJSON(w, http.StatusCreated, user)
writeError(w, http.StatusBadRequest, "invalid email")
```

## Уровень 9: про регистрацию в RegisterRoutes

```go
func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    mux.HandleFunc("GET /hello", h.hello)
    mux.HandleFunc("POST /register", h.register)
}
```

Каждый новый эндпоинт добавляется здесь. В модулях 3-5 ты добавишь ещё несколько строк.

## Уровень 10: про подключение к БД в main

```go
import (
    _ "github.com/jackc/pgx/v5/stdlib"  // регистрация драйвера
)

func main() {
    cfg, err := config.Load()
    // ...

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

    log.Println("Connected to PostgreSQL")

    // Создаём зависимости с РЕАЛЬНЫМ db
    userRepo := storage.NewUserRepository(db)
    // ...
}
```

Заметь импорт драйвера через `_` — без него `sql.Open("pgx", ...)` не найдёт драйвер.

## Если совсем тупик

Скажи Claude конкретно: «модуль 2 проекта 7, [конкретная проблема]». Самые частые ситуации:

- **«unknown driver "pgx"»** → забыл `_ "github.com/jackc/pgx/v5/stdlib"`
- **«password_hash появляется в ответе»** → проверь тег `json:"-"` (именно дефис, не пустая строка)
- **«email already exists, но errors.Is(err, ErrEmailExists) = false»** → ошибка в `isUniqueViolation` или забыл вернуть `ErrEmailExists`
- **«panic: runtime error: nil pointer»** → `db` всё ещё `nil`, забыл обновить `main.go`
- **«password too long»** → bcrypt принимает максимум 72 байта, но обычно молча обрезает. Если падает — версия bcrypt новая, проверь документацию.
