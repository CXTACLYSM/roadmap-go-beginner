# Модуль 3. JWT и middleware — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какая библиотека для JWT в Go?
2. Какой алгоритм подписи использовать (HS256, RS256)?
3. Какие три claim-а минимально нужны (`sub`, `exp`, `iat`)?
4. Какой формат у HTTP-заголовка `Authorization`?
5. Как передать значение из middleware в handler?

## Уровень 2: про jwt.NewWithClaims

```go
import "github.com/golang-jwt/jwt/v5"

claims := jwt.MapClaims{
    "sub": userID,
    "exp": time.Now().Add(24 * time.Hour).Unix(),
    "iat": time.Now().Unix(),
}
token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
tokenStr, err := token.SignedString([]byte(s.secret))
```

Несколько моментов:

- **`jwt.MapClaims`** — это `map[string]interface{}`, удобно для прототипов
- **`Unix()`** возвращает Unix-секунды (int64), это стандарт для JWT
- **`SignedString([]byte(secret))`** возвращает финальную строку токена
- **Секрет должен быть `[]byte`**, не строка

## Уровень 3: про jwt.Parse

```go
parsed, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
    }
    return []byte(s.secret), nil
})
```

Callback вызывается **во время парсинга** и должен:

1. **Проверить алгоритм подписи** — type assertion `t.Method.(*jwt.SigningMethodHMAC)`. Это **обязательно** для безопасности.
2. **Вернуть ключ** — `[]byte(s.secret)` для HMAC

Без проверки алгоритма атакующий может прислать токен с `alg: none` и обойти проверку.

После парсинга:

```go
if !parsed.Valid {
    return 0, ErrInvalidToken
}

claims, ok := parsed.Claims.(jwt.MapClaims)
if !ok {
    return 0, ErrInvalidToken
}

sub, ok := claims["sub"].(float64)  // ← float64, не int64!
if !ok {
    return 0, ErrInvalidToken
}

return int64(sub), nil
```

## Уровень 4: про float64 в JSON

В JSON все числа — `float64`. Когда ты сохраняешь `sub: 1` (`int64`) и потом парсишь обратно, получаешь `float64(1)`. Type assertion должен быть `claims["sub"].(float64)`, а не `(int64)`.

Если ты используешь **custom claims** (свою структуру вместо `MapClaims`), эта проблема решается через теги:

```go
type AppClaims struct {
    UserID int64 `json:"sub,string"`
    jwt.RegisteredClaims
}
```

Но `MapClaims` проще для модуля 3.

## Уровень 5: про middleware-функцию

Стандартная сигнатура:

```go
func (h *Handler) authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 1. Проверяем заголовок
        // 2. Если плохо — пишем ответ и НЕ вызываем next
        // 3. Если ОК — модифицируем контекст и вызываем next
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

Несколько моментов:

- **Принимает `next http.Handler`** — следующий обработчик
- **Возвращает `http.Handler`** — новый, который оборачивает next
- **`http.HandlerFunc`** — конвертер из функции в `http.Handler`
- **`next.ServeHTTP(w, r)`** — вызов следующего обработчика
- **`r.WithContext(ctx)`** — новый запрос с новым контекстом

## Уровень 6: про Bearer token парсинг

```go
authHeader := r.Header.Get("Authorization")
if authHeader == "" {
    writeError(w, http.StatusUnauthorized, "missing authorization header")
    return
}

parts := strings.SplitN(authHeader, " ", 2)
if len(parts) != 2 || parts[0] != "Bearer" {
    writeError(w, http.StatusUnauthorized, "invalid authorization format")
    return
}
tokenStr := parts[1]
```

Формат: `Bearer <token>` — две части, разделённые пробелом. `SplitN(s, " ", 2)` режет максимум на 2 части, что важно: если токен содержит пробел (не должен, но мало ли), он не разрежется.

## Уровень 7: про context.WithValue

```go
import "context"

type contextKey string

const userIDKey contextKey = "user_id"

// В middleware:
ctx := context.WithValue(r.Context(), userIDKey, userID)
next.ServeHTTP(w, r.WithContext(ctx))

// В handler:
userID, ok := r.Context().Value(userIDKey).(int64)
if !ok {
    // что-то не так
}
```

⚠️ **КРИТИЧНО**: используй **свой тип** для ключа, не голую строку:

```go
// ПРАВИЛЬНО
type contextKey string
const userIDKey contextKey = "user_id"
ctx.Value(userIDKey)

// НЕПРАВИЛЬНО (компилируется, но опасно)
ctx.Value("user_id")  // голая строка
```

Зачем? Потому что контекст — общий объект. Если два пакета используют ключ `"user_id"` (голую строку) — они **перетрут** друг друга. С собственным типом этого не случится, потому что `contextKey("user_id")` ≠ `string("user_id")`.

Линтер `staticcheck` ловит это правило.

## Уровень 8: про userIDFromContext helper

```go
func userIDFromContext(ctx context.Context) (int64, bool) {
    id, ok := ctx.Value(userIDKey).(int64)
    return id, ok
}
```

Это **обёртка** над `ctx.Value`. Зачем:

- Чище в handler: `userID, ok := userIDFromContext(r.Context())` vs `userID, ok := r.Context().Value(userIDKey).(int64)`
- Скрывает детали (тип ключа, тип значения)
- Если в будущем поменяешь тип — поправишь в одном месте

## Уровень 9: про регистрацию защищённых роутов

```go
func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    // Публичные — HandleFunc для функций
    mux.HandleFunc("GET /hello", h.hello)
    mux.HandleFunc("POST /register", h.register)
    mux.HandleFunc("POST /login", h.login)

    // Защищённые — Handle для http.Handler (middleware возвращает Handler)
    mux.Handle("GET /me", h.authMiddleware(http.HandlerFunc(h.me)))
}
```

Заметь:
- **`HandleFunc`** для голых функций
- **`Handle`** для типа `http.Handler` (что возвращает middleware)
- **`http.HandlerFunc(h.me)`** — конвертер: оборачивает функцию в тип `http.Handler` для прохождения через middleware

## Уровень 10: про защиту от user enumeration

Главная мысль: **`/login` всегда должен возвращать одинаковый ответ** для случаев «нет пользователя» и «неправильный пароль».

```go
user, err := h.users.GetByEmail(r.Context(), input.Email)
if err != nil {
    if errors.Is(err, storage.ErrUserNotFound) {
        // НЕ говорим "user not found", иначе атакующий узнает, какие email есть
        writeError(w, http.StatusUnauthorized, "invalid credentials")
        return
    }
    // ... другие ошибки → 500
}

if err := h.auth.CheckPassword(user.PasswordHash, input.Password); err != nil {
    if errors.Is(err, auth.ErrInvalidPassword) {
        // ТО ЖЕ САМОЕ сообщение
        writeError(w, http.StatusUnauthorized, "invalid credentials")
        return
    }
    // ...
}
```

Сообщение в обоих случаях — **точно одно и то же**: `"invalid credentials"`. И статус один — 401.

## Если совсем тупик

Скажи Claude конкретно: «модуль 3 проекта 7, [конкретная проблема]». Самые частые ситуации:

- **«token contains an invalid number of segments»** → токен битый или не передаётся целиком
- **«signature is invalid»** → разные секреты в `GenerateToken` и `ValidateToken`
- **«token is expired»** → нормально, токен истёк, перелогинься
- **«claims["sub"] is float64»** → ты сделал type assertion в `int64`, нужно через `float64` → `int64`
- **«context value is nil»** → middleware не вызвал, или ты используешь `string` вместо своего типа для ключа
- **«method assertion failed»** → проверь, что в callback ты делаешь `t.Method.(*jwt.SigningMethodHMAC)`
