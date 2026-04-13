# Модуль 2. Users и регистрация — теория

## Контекст

В модуле 1 ты построил **скелет** проекта, но он пока ничего не делает кроме `Hello, blog!`. В этом модуле появится **первая реальная функциональность**: регистрация пользователей.

Ты:

- Создашь таблицу `users` в БД через миграцию
- Подключишь БД из `main.go` и передашь в `UserRepository`
- Узнаешь, как **правильно хранить пароли** через bcrypt
- Реализуешь POST `/register` с валидацией
- Поймёшь, **почему** хранение паролей в plain text — катастрофа

К концу модуля любой клиент сможет создать аккаунт через `curl`, и пароль в БД будет захеширован.

## Ядро 1: почему нельзя хранить пароли в plain text

Это **первый закон** работы с паролями. Никаких исключений.

Если ты хранишь пароли как есть (`password: "secret123"`), то:

1. **Любой**, кто получит доступ к БД (бэкап, SQL injection, кража диска), увидит **все** пароли пользователей
2. **Пользователи переиспользуют пароли** — Лев использовал `secret123` и в твоём блоге, и в Gmail. Утечка твоей БД = утечка чужих аккаунтов
3. **Логи могут случайно содержать пароли** — если в логах SQL-запросов виден `INSERT INTO users (..., password) VALUES ('secret123')`, это утечка

История полна катастроф: LinkedIn (2012), Yahoo (2013), Adobe (2013) — десятки миллионов паролей утекли. У всех этих компаний была **плохая** работа с паролями.

**Правило:** в БД лежит **не пароль**, а **результат односторонней функции от пароля** — хеш. По хешу нельзя восстановить пароль. При проверке мы хешируем введённый пароль и сравниваем с сохранённым.

## Ядро 2: какие хеши подходят, а какие нет

Не любая хеш-функция подходит. Главные плохие варианты:

- **MD5, SHA1** — слишком быстрые. Атакующий может перебрать миллиарды вариантов в секунду. Категорически нельзя.
- **SHA256, SHA512** без соли — тоже быстрые. Тоже нельзя.

Хорошие варианты:

- **bcrypt** — старый, проверенный, медленный (намеренно). Работает с 1999 года, до сих пор стандарт.
- **argon2** — современный, выиграл Password Hashing Competition в 2015. Лучше bcrypt теоретически, но требует больше внимания при настройке параметров.
- **scrypt** — между bcrypt и argon2.

**Для нашего проекта используем bcrypt.** Это:

- Стандартный выбор для Go (`golang.org/x/crypto/bcrypt`)
- Простой API
- Достаточно надёжный
- В большинстве бэкендов в Go ты увидишь именно его

Argon2 — для более продвинутых проектов или для криптографически более чувствительных данных. Принципы те же.

## Ядро 3: что делает bcrypt

bcrypt — это **медленный хеш с встроенной солью**. Что это значит:

### Соль

**Соль** — это случайная строка, добавляемая к паролю **перед** хешированием. У каждого пользователя — своя соль. Зачем:

- Без соли два пользователя с одинаковым паролем имеют одинаковый хеш. Атакующий может построить **rainbow table** — таблицу `(хеш → пароль)` для всех популярных паролей. Один lookup в таблице и пароль раскрыт.
- С разной солью у двух одинаковых паролей **разные** хеши. Rainbow tables становятся бесполезны — для каждой соли нужна отдельная таблица.

В bcrypt **соль генерируется автоматически** при каждом вызове `GenerateFromPassword`. Тебе не нужно об этом думать.

### Медленность

bcrypt **намеренно медленный**. Один вызов хеша занимает ~100 миллисекунд (зависит от cost parameter). Это:

- **Бесит атакующего** — вместо миллиарда попыток в секунду он делает 10
- **Не мешает обычному использованию** — один вызов на регистрацию или логин незаметен пользователю

Параметр **cost** регулирует медленность. Стандартный cost = 10 (это `bcrypt.DefaultCost`). Каждое увеличение на 1 удваивает время.

### Формат хеша

Результат bcrypt — строка вида:

```
$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
```

Это самодостаточная строка, которая содержит:

- **`$2a$`** — алгоритм (bcrypt версии 2a)
- **`$10$`** — cost = 10
- **`N9qo8uLOickgx2ZMRZoMye`** — соль (22 символа)
- **`IjZAgcfl7p92ldGxad68LJZdL17lhWy`** — собственно хеш

Когда ты потом хочешь проверить пароль — bcrypt сам извлекает из этой строки соль и cost и применяет тот же алгоритм. Поэтому **тебе не нужно отдельно хранить соль**.

## Ядро 4: API bcrypt в Go

```go
import "golang.org/x/crypto/bcrypt"

// Хеширование при регистрации
hash, err := bcrypt.GenerateFromPassword([]byte("password123"), bcrypt.DefaultCost)
if err != nil {
    return err
}
// hash — это []byte, конвертируем в string для хранения в БД
hashStr := string(hash)

// Проверка при логине
err := bcrypt.CompareHashAndPassword([]byte(hashStr), []byte("password123"))
if err == nil {
    // пароль правильный
} else if errors.Is(err, bcrypt.ErrMismatchedHashAndPassword) {
    // пароль неправильный
} else {
    // другая ошибка (повреждённый хеш и т.д.)
}
```

Два метода:

- **`GenerateFromPassword(password, cost) ([]byte, error)`** — хеширует пароль
- **`CompareHashAndPassword(hash, password) error`** — проверяет. `nil` = совпало, `ErrMismatchedHashAndPassword` = не совпало

`bcrypt.DefaultCost` сейчас 10. Можно поставить выше (12, 14), если хочешь больше безопасности — но регистрация и логин станут медленнее.

## Ядро 5: схема таблицы users

```sql
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);
```

Несколько важных моментов:

1. **`email TEXT NOT NULL UNIQUE`** — `UNIQUE` гарантирует, что не будет двух пользователей с одним email. Если попытаешься вставить дубликат — PostgreSQL вернёт ошибку `unique_violation`.

2. **`password_hash TEXT NOT NULL`** — храним именно **хеш**, не пароль. Имя поля прямо указывает: «здесь хеш, не plain text». Никогда не называй поле `password`.

3. **`UNIQUE` автоматически создаёт индекс** на колонке. Поэтому отдельный `CREATE INDEX idx_users_email` **избыточен** — `UNIQUE INDEX` уже есть. Я оставил его в примере для ясности; в реальной миграции можешь убрать.

4. **`created_at TIMESTAMPTZ`** — стандартное поле, позволяет понять, когда зарегистрирован пользователь.

## Ядро 6: что НЕ должно попасть в API ответ

Когда ты возвращаешь пользователя в JSON, **`password_hash` НИКОГДА не должен попадать в ответ**. Никогда. Даже если кажется безобидным.

Решение через JSON-теги:

```go
type User struct {
    ID           int64     `json:"id"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`            // ← `-` означает "не сериализовать"
    CreatedAt    time.Time `json:"created_at"`
}
```

`json:"-"` — это специальное значение для тега. Оно говорит `encoding/json`: «это поле игнорируется при сериализации и десериализации».

Тогда:

```go
user := &User{ID: 1, Email: "lev@x.com", PasswordHash: "$2a$10$..."}
data, _ := json.Marshal(user)
// {"id":1,"email":"lev@x.com","created_at":"..."}
// PasswordHash отсутствует
```

**Это страховка**. Даже если ты случайно где-то верёшь весь объект — пароль не утечёт.

## Ядро 7: тип User в domain

```go
// internal/domain/user.go
package domain

import "time"

type User struct {
    ID           int64     `json:"id"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`
    CreatedAt    time.Time `json:"created_at"`
}
```

Это — **единый** тип для всего проекта. Используется в `storage` (туда писать), в `auth` (передавать), в `api` (отвечать клиенту).

`PasswordHash` — приватный для JSON, но **публичный для Go** (с большой буквы). Это нужно, чтобы `storage.Create` мог его записать и `storage.GetByEmail` мог прочитать.

## Ядро 8: репозиторий — методы

```go
// internal/storage/user.go
package storage

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "blog/internal/domain"
)

var ErrUserNotFound = errors.New("user not found")
var ErrEmailExists = errors.New("email already exists")

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(ctx context.Context, email, passwordHash string) (*domain.User, error) {
    var u domain.User
    err := r.db.QueryRowContext(ctx, `
        INSERT INTO users (email, password_hash)
        VALUES ($1, $2)
        RETURNING id, email, password_hash, created_at
    `, email, passwordHash).Scan(&u.ID, &u.Email, &u.PasswordHash, &u.CreatedAt)

    if err != nil {
        // Здесь — самая важная часть.
        // PostgreSQL возвращает специфичную ошибку для дубликата UNIQUE.
        // Мы её ловим и заменяем на свою доменную ошибку.
        if isUniqueViolation(err) {
            return nil, ErrEmailExists
        }
        return nil, fmt.Errorf("create user: %w", err)
    }

    return &u, nil
}

func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    var u domain.User
    err := r.db.QueryRowContext(ctx, `
        SELECT id, email, password_hash, created_at
        FROM users
        WHERE email = $1
    `, email).Scan(&u.ID, &u.Email, &u.PasswordHash, &u.CreatedAt)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("get user: %w", err)
    }

    return &u, nil
}
```

Несколько важных моментов:

1. **Sentinel ошибки** — `ErrUserNotFound`, `ErrEmailExists` — объявлены как пакетные переменные. Внешний код проверяет через `errors.Is`. Это **чище**, чем парсить сообщения об ошибках.

2. **`isUniqueViolation(err)`** — функция, которая распознаёт ошибку **дубликата UNIQUE** из PostgreSQL. Реализация:

   ```go
   import "github.com/jackc/pgx/v5/pgconn"

   func isUniqueViolation(err error) bool {
       var pgErr *pgconn.PgError
       if errors.As(err, &pgErr) {
           return pgErr.Code == "23505"  // unique_violation
       }
       return false
   }
   ```

   `errors.As` — это «дай мне ошибку этого типа из цепочки обёрток». PostgreSQL коды ошибок — пятизначные строки, `23505` — это unique violation. Это работает с pgx-драйвером.

3. **Контексты используются** — `QueryRowContext`. Это привычка из Проекта 6.

## Ядро 9: bcrypt в auth-сервисе

```go
// internal/auth/service.go
package auth

import (
    "errors"

    "golang.org/x/crypto/bcrypt"
)

var ErrInvalidPassword = errors.New("invalid password")

type Service struct {
    secret string
}

func NewService(secret string) *Service {
    return &Service{secret: secret}
}

func (s *Service) HashPassword(password string) (string, error) {
    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return "", err
    }
    return string(hash), nil
}

func (s *Service) CheckPassword(hash, password string) error {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    if err != nil {
        if errors.Is(err, bcrypt.ErrMismatchedHashAndPassword) {
            return ErrInvalidPassword
        }
        return err
    }
    return nil
}
```

Это первые **реальные методы** в `auth.Service`. Поле `secret` пока не используется (понадобится в модуле 3 для JWT).

## Ядро 10: handler регистрации

```go
// internal/api/users.go
package api

import (
    "encoding/json"
    "errors"
    "log"
    "net/http"
    "regexp"
    "strings"

    "blog/internal/storage"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

func (h *Handler) register(w http.ResponseWriter, r *http.Request) {
    var input struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "invalid json")
        return
    }

    // Валидация email
    input.Email = strings.TrimSpace(strings.ToLower(input.Email))
    if !emailRegex.MatchString(input.Email) {
        writeError(w, http.StatusBadRequest, "invalid email")
        return
    }

    // Валидация пароля
    if len(input.Password) < 8 {
        writeError(w, http.StatusBadRequest, "password must be at least 8 characters")
        return
    }

    // Хешируем пароль
    hash, err := h.auth.HashPassword(input.Password)
    if err != nil {
        log.Printf("hash password: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    // Создаём пользователя
    user, err := h.users.Create(r.Context(), input.Email, hash)
    if err != nil {
        if errors.Is(err, storage.ErrEmailExists) {
            writeError(w, http.StatusConflict, "email already exists")
            return
        }
        log.Printf("create user: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    writeJSON(w, http.StatusCreated, user)
}

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

Несколько важных моментов:

1. **Email нормализуется**: `TrimSpace + ToLower`. Это значит `Lev@Example.COM` и `lev@example.com` — это **один** пользователь. Это стандартная практика.

2. **Email валидируется регулярным выражением**. Это базовая валидация. Полная валидация email — сложная (RFC 5322), но регулярка покрывает 99% реальных случаев.

3. **Пароль валидируется по длине**. Минимум 8 символов. В реальном проекте можно добавить проверки на сложность (цифры, спецсимволы), но они часто бесят пользователей и **не сильно увеличивают безопасность** (длинная фраза-парольник лучше короткого `P@ssw0rd!`).

4. **`ErrEmailExists` → 409 Conflict**. Это правильный статус для «ресурс уже существует». Не 400, не 500.

5. **`writeJSON` и `writeError` — helpers**. Это уберёт дублирование при следующих эндпоинтах.

6. **`r.Context()`** — пробрасываем контекст HTTP-запроса в репозиторий.

## Ядро 11: обновление main и подключение БД

```go
// cmd/server/main.go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"

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

    userRepo := storage.NewUserRepository(db)
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

И в `RegisterRoutes`:

```go
func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    mux.HandleFunc("GET /hello", h.hello)
    mux.HandleFunc("POST /register", h.register)
}
```

## Вопросы на понимание

1. Что произойдёт, если хранить пароли в plain text и БД утечёт?
2. Что такое **соль** в хешировании паролей? Зачем она?
3. Почему bcrypt **намеренно медленный**?
4. Какие два главных метода bcrypt в Go?
5. Что такое cost parameter и какой стандартный?
6. Зачем `json:"-"` на поле `PasswordHash`?
7. Какая ошибка приходит от PostgreSQL при дубликате UNIQUE? Какой код?
8. Что такое **sentinel error** и как её ловить через `errors.Is`?
9. Какой HTTP-статус возвращать при «email уже занят»?
10. Зачем нормализовать email через `ToLower` перед сохранением?
