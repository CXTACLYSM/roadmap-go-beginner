# Модуль 3. JWT и middleware — теория

## Контекст

В модуле 2 пользователь может зарегистрироваться. Но что **дальше**? Как сервер понимает, что следующий запрос идёт от **этого** пользователя, а не от другого?

В этом модуле ты добавишь **аутентификацию по токену**:

- Реализуешь POST `/login`, который проверяет пароль и выдаёт **JWT-токен**
- Поймёшь, что такое JWT и как оно устроено
- Реализуешь **middleware** — функцию, которая оборачивает HTTP-обработчики и проверяет токен **до** их вызова
- Передашь `user_id` через **request context** от middleware в handler
- Реализуешь GET `/me` — защищённый эндпоинт, который возвращает данные текущего пользователя

После этого модуля у тебя будет полная инфраструктура для multi-user сервиса. В модуле 4 ты добавишь посты, и каждый пост будет привязан к **тому**, кто его создал.

## Ядро 1: что такое JWT

**JWT** (JSON Web Token) — это **подписанная** строка, содержащая JSON-данные. Когда сервер её выдаёт, он подписывает своим секретным ключом. Когда получает обратно — проверяет подпись. Если подпись валидна, сервер **доверяет** содержимому.

Пример JWT:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxIiwiZXhwIjoxNzQzMjcwNDAwfQ.4n2GxR3FjQDx2qjZjLZ4dWP_zBFcDi_aNgcdjJpkPSI
```

Это три части, разделённые точками:

```
header  .  payload  .  signature
```

### Header

Base64-кодированный JSON:

```json
{"alg":"HS256","typ":"JWT"}
```

`alg` — алгоритм подписи (`HS256` = HMAC-SHA256). `typ` — тип (всегда `JWT`).

### Payload (claims)

Тоже base64-кодированный JSON, содержит **claims** — утверждения о пользователе:

```json
{
  "sub": "1",
  "exp": 1743270400,
  "iat": 1743184000
}
```

- **`sub`** (subject) — кому принадлежит токен. Обычно user ID.
- **`exp`** (expiration) — Unix-время, когда токен истечёт
- **`iat`** (issued at) — когда выдан
- **Любые свои поля** — например, `username`, `role`, `email`

### Signature

HMAC от `header.payload` с секретным ключом сервера. Если кто-то изменит header или payload, подпись не сойдётся.

## Ядро 2: важное про JWT

**Главные мысли**, которые путают новички:

### 1. JWT — это НЕ шифрование

Содержимое payload — **base64**, а не зашифровано. Любой может его прочитать. Открой [jwt.io](https://jwt.io), вставь свой токен — увидишь все claims.

**Не клади в JWT секретные данные.** Только то, что не страшно показать пользователю: его id, роль, имя.

### 2. Подпись защищает от **изменения**, не от **чтения**

Атакующий может прочитать payload, но не может его изменить (без секретного ключа сервера). Если изменит — сервер увидит, что подпись неверна.

### 3. JWT не отзываемый «из коробки»

Если ты выдал токен с `exp` через 24 часа, а через час пользователь сменил пароль или ты заблокировал аккаунт — старый токен **всё ещё валиден** до истечения. Чтобы отзывать — нужны дополнительные механизмы (blacklist в Redis, refresh tokens, короткий exp).

Для нашего проекта мы выдаём токены на 24 часа без отзыва. Для production — нужно подумать.

### 4. JWT часто **переоценен**

Для большинства приложений **сессии в Redis** (или даже в БД) проще и безопаснее. JWT хорош, когда:
- Stateless требование (балансировка без sticky sessions)
- Несколько сервисов проверяют один и тот же токен (не нужна общая сессионная БД)
- Mobile/SPA, где cookies неудобны

В нашем проекте JWT — потому что это **отраслевой стандарт** и ты должен уметь работать с ним.

## Ядро 3: библиотека golang-jwt

Стандарт для Go — `github.com/golang-jwt/jwt/v5`. Установка:

```bash
go get github.com/golang-jwt/jwt/v5
```

API в общих чертах:

```go
import "github.com/golang-jwt/jwt/v5"

// Создание токена
claims := jwt.MapClaims{
    "sub": userID,
    "exp": time.Now().Add(24 * time.Hour).Unix(),
    "iat": time.Now().Unix(),
}
token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
tokenStr, err := token.SignedString([]byte(secret))

// Парсинг и проверка
parsedToken, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
    }
    return []byte(secret), nil
})

if err != nil || !parsedToken.Valid {
    // токен невалиден
}

claims := parsedToken.Claims.(jwt.MapClaims)
sub := claims["sub"].(string)  // или float64, зависит от того, как сохранил
```

Несколько важных моментов:

1. **`SigningMethodHS256`** — симметричный алгоритм. Один секретный ключ и для подписи, и для проверки. Для нашего случая (один сервер выдаёт и проверяет) — оптимально.

2. **`callback в jwt.Parse`** — это безопасность. Без проверки `t.Method.(*jwt.SigningMethodHMAC)` атакующий может прислать токен с алгоритмом `none` и обойти проверку. Это **известная атака**, и `golang-jwt` требует от тебя явно проверить алгоритм.

3. **`MapClaims` vs custom claims** — `MapClaims` это `map[string]interface{}`, удобно для прототипов. Для production обычно используют **свой тип** через интерфейс `jwt.Claims`. Для нашего проекта — `MapClaims` достаточно.

## Ядро 4: расширение auth.Service

```go
// internal/auth/service.go
package auth

import (
    "errors"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "golang.org/x/crypto/bcrypt"
)

var (
    ErrInvalidPassword = errors.New("invalid password")
    ErrInvalidToken    = errors.New("invalid token")
)

const tokenLifetime = 24 * time.Hour

type Service struct {
    secret string
}

func NewService(secret string) *Service {
    return &Service{secret: secret}
}

// HashPassword и CheckPassword — из модуля 2

func (s *Service) GenerateToken(userID int64) (string, error) {
    claims := jwt.MapClaims{
        "sub": userID,
        "exp": time.Now().Add(tokenLifetime).Unix(),
        "iat": time.Now().Unix(),
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    tokenStr, err := token.SignedString([]byte(s.secret))
    if err != nil {
        return "", fmt.Errorf("sign token: %w", err)
    }
    return tokenStr, nil
}

func (s *Service) ValidateToken(tokenStr string) (int64, error) {
    parsed, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return []byte(s.secret), nil
    })
    if err != nil {
        return 0, fmt.Errorf("%w: %v", ErrInvalidToken, err)
    }
    if !parsed.Valid {
        return 0, ErrInvalidToken
    }

    claims, ok := parsed.Claims.(jwt.MapClaims)
    if !ok {
        return 0, ErrInvalidToken
    }

    // sub приходит как float64 (стандарт JSON), нужна конвертация
    sub, ok := claims["sub"].(float64)
    if !ok {
        return 0, ErrInvalidToken
    }

    return int64(sub), nil
}
```

⚠️ **Тонкость с `sub` как float64**: при сериализации JSON число становится `float64`, а не `int64`. Это **известная странность** Go's `encoding/json`. При парсинге через `MapClaims` придётся приводить через `float64` → `int64`. В custom claims-структурах с `int64` тегами эта проблема решается, но `MapClaims` так работает.

## Ядро 5: handler логина

```go
// internal/api/users.go (продолжение)

func (h *Handler) login(w http.ResponseWriter, r *http.Request) {
    var input struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "invalid json")
        return
    }

    input.Email = strings.TrimSpace(strings.ToLower(input.Email))

    // Получаем пользователя
    user, err := h.users.GetByEmail(r.Context(), input.Email)
    if err != nil {
        if errors.Is(err, storage.ErrUserNotFound) {
            // ВАЖНО: возвращаем такое же сообщение, как для неправильного пароля.
            // Иначе по разнице в ответах атакующий может определить, какие email зарегистрированы.
            writeError(w, http.StatusUnauthorized, "invalid credentials")
            return
        }
        log.Printf("get user: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    // Проверяем пароль
    if err := h.auth.CheckPassword(user.PasswordHash, input.Password); err != nil {
        if errors.Is(err, auth.ErrInvalidPassword) {
            writeError(w, http.StatusUnauthorized, "invalid credentials")
            return
        }
        log.Printf("check password: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    // Генерируем токен
    token, err := h.auth.GenerateToken(user.ID)
    if err != nil {
        log.Printf("generate token: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    writeJSON(w, http.StatusOK, map[string]string{"token": token})
}
```

⚠️ **Принципиальная безопасность:** «пользователь не найден» и «пароль неверный» **возвращают одинаковое сообщение** «invalid credentials». Иначе атакующий может через POST `/login` перебрать миллионы email-адресов и узнать, какие зарегистрированы у тебя. Это называется **user enumeration**.

## Ядро 6: что такое middleware

**Middleware** — это функция, которая **оборачивает** HTTP-обработчик. Она получает `(w, r)`, делает свою работу, и **либо** вызывает следующий обработчик, **либо** возвращает ответ напрямую.

Стандартная сигнатура middleware в Go:

```go
func Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // что-то делаем до
        next.ServeHTTP(w, r)
        // что-то делаем после
    })
}
```

Это **функция, возвращающая функцию**. Принимает «следующий» обработчик, возвращает новый.

Использование:

```go
mux.Handle("GET /me", Middleware(http.HandlerFunc(h.me)))
```

Middleware теперь оборачивает `h.me`. Когда придёт запрос на `/me`, сначала выполнится код middleware, потом — `h.me`.

Можно **цепочкой**:

```go
mux.Handle("GET /me", LoggingMiddleware(AuthMiddleware(http.HandlerFunc(h.me))))
```

Сначала LoggingMiddleware, потом AuthMiddleware, потом сам handler. Это паттерн **chain of responsibility**.

## Ядро 7: AuthMiddleware

```go
// internal/api/middleware.go
package api

import (
    "context"
    "log"
    "net/http"
    "strings"

    "blog/internal/auth"
)

type contextKey string

const userIDKey contextKey = "user_id"

func (h *Handler) authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 1. Получаем заголовок Authorization
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            writeError(w, http.StatusUnauthorized, "missing authorization header")
            return
        }

        // 2. Должен быть формат "Bearer <token>"
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            writeError(w, http.StatusUnauthorized, "invalid authorization format")
            return
        }
        tokenStr := parts[1]

        // 3. Валидируем токен
        userID, err := h.auth.ValidateToken(tokenStr)
        if err != nil {
            writeError(w, http.StatusUnauthorized, "invalid token")
            return
        }

        // 4. Кладём userID в контекст и передаём дальше
        ctx := context.WithValue(r.Context(), userIDKey, userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func userIDFromContext(ctx context.Context) (int64, bool) {
    id, ok := ctx.Value(userIDKey).(int64)
    return id, ok
}
```

Несколько важных моментов:

1. **`type contextKey string`** — свой тип для ключа в контексте. Это **обязательная практика** в Go: использовать **свой** тип, а не голую строку. Иначе можно случайно перетереть ключ другого пакета.

2. **Формат `Authorization: Bearer <token>`** — стандарт HTTP (RFC 6750). Любой клиент шлёт токены так.

3. **`context.WithValue(ctx, key, value)`** — создаёт новый контекст с дополнительным значением. Контексты **immutable** — каждый `WithValue` возвращает новый.

4. **`r.WithContext(ctx)`** — создаёт новый запрос с обновлённым контекстом. Передаём его в `next.ServeHTTP`.

5. **`userIDFromContext` helper** — извлекает userID из контекста. Возвращает `(int64, bool)`. Использование: `userID, ok := userIDFromContext(r.Context())`.

## Ядро 8: handler /me

```go
func (h *Handler) me(w http.ResponseWriter, r *http.Request) {
    userID, ok := userIDFromContext(r.Context())
    if !ok {
        writeError(w, http.StatusInternalServerError, "user id not found in context")
        return
    }

    user, err := h.users.GetByID(r.Context(), userID)
    if err != nil {
        if errors.Is(err, storage.ErrUserNotFound) {
            writeError(w, http.StatusNotFound, "user not found")
            return
        }
        log.Printf("get user: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    writeJSON(w, http.StatusOK, user)
}
```

Заметь: **сам handler ничего не знает про токены**. Он просто берёт `userID` из контекста и работает дальше. Middleware сделал всю грязную работу.

Это **главное преимущество middleware**: каждый защищённый эндпоинт пишется так, **как будто пользователь уже известен**.

⚠️ **Нужно добавить метод `GetByID`** в репозиторий. Это симметрично `GetByEmail`:

```go
func (r *UserRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    var u domain.User
    err := r.db.QueryRowContext(ctx, `
        SELECT id, email, password_hash, created_at FROM users WHERE id = $1
    `, id).Scan(&u.ID, &u.Email, &u.PasswordHash, &u.CreatedAt)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("get user: %w", err)
    }
    return &u, nil
}
```

## Ядро 9: регистрация роутов с middleware

```go
func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    // Публичные
    mux.HandleFunc("GET /hello", h.hello)
    mux.HandleFunc("POST /register", h.register)
    mux.HandleFunc("POST /login", h.login)

    // Защищённые — оборачиваем в middleware
    mux.Handle("GET /me", h.authMiddleware(http.HandlerFunc(h.me)))
}
```

Заметь:

- **`mux.HandleFunc(path, fn)`** для функций
- **`mux.Handle(path, handler)`** для `http.Handler` (а middleware возвращает именно `http.Handler`)
- **`http.HandlerFunc(h.me)`** — конвертер: оборачивает функцию в тип `http.Handler`

## Ядро 10: тестовый сценарий

```bash
# Регистрация
$ curl -X POST http://localhost:8080/register -H "Content-Type: application/json" \
    -d '{"email":"lev@example.com","password":"strong-password-123"}'
{"id":1,"email":"lev@example.com","created_at":"..."}

# Логин
$ curl -X POST http://localhost:8080/login -H "Content-Type: application/json" \
    -d '{"email":"lev@example.com","password":"strong-password-123"}'
{"token":"eyJhbGciOiJIUzI1NiIs..."}

# Получаем токен
$ TOKEN="eyJhbGciOiJIUzI1NiIs..."

# /me с токеном — работает
$ curl http://localhost:8080/me -H "Authorization: Bearer $TOKEN"
{"id":1,"email":"lev@example.com","created_at":"..."}

# /me без токена — 401
$ curl -i http://localhost:8080/me
HTTP/1.1 401 Unauthorized
{"error":"missing authorization header"}

# /me с битым токеном — 401
$ curl -i http://localhost:8080/me -H "Authorization: Bearer broken"
HTTP/1.1 401 Unauthorized
{"error":"invalid token"}
```

После этого модуля у тебя есть **полный auth flow**. В модулях 4-5 ты применишь его к постам и комментариям.

## Вопросы на понимание

1. Что такое JWT и из каких трёх частей состоит?
2. Почему JWT — это **не шифрование**?
3. Что такое **claims** и какие стандартные ты знаешь (`sub`, `exp`, `iat`)?
4. Какой алгоритм мы используем для подписи и почему?
5. Зачем callback в `jwt.Parse` проверяет `t.Method`?
6. Что такое **user enumeration** и как мы от него защищаемся в `/login`?
7. Что такое **middleware** в HTTP-сервере?
8. Зачем нужен **свой тип** `contextKey` для ключа в контексте?
9. Что делает `context.WithValue`?
10. Чем `mux.HandleFunc` отличается от `mux.Handle`?
