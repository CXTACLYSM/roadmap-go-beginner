# Модуль 3. JWT и middleware — условие

## Задание

Реализовать `auth.GenerateToken` и `auth.ValidateToken`, эндпоинт `POST /login` с проверкой пароля и выдачей JWT, **middleware** для защищённых эндпоинтов, эндпоинт `GET /me`. После этого у тебя будет полный auth flow.

## Что должно произойти

```bash
# Регистрация
$ curl -X POST http://localhost:8080/register -H "Content-Type: application/json" \
    -d '{"email":"lev@example.com","password":"strong-password-123"}'
{"id":1,"email":"lev@example.com","created_at":"..."}

# Логин
$ curl -X POST http://localhost:8080/login -H "Content-Type: application/json" \
    -d '{"email":"lev@example.com","password":"strong-password-123"}'
{"token":"eyJhbGciOiJIUzI1NiIs..."}

# Логин с неправильным паролем
$ curl -i -X POST http://localhost:8080/login -H "Content-Type: application/json" \
    -d '{"email":"lev@example.com","password":"wrong"}'
HTTP/1.1 401 Unauthorized
{"error":"invalid credentials"}

# Логин с несуществующим email
$ curl -i -X POST http://localhost:8080/login -H "Content-Type: application/json" \
    -d '{"email":"nobody@example.com","password":"anything"}'
HTTP/1.1 401 Unauthorized
{"error":"invalid credentials"}
# ВАЖНО: то же сообщение, что для неправильного пароля!

# /me с валидным токеном
$ TOKEN="eyJhbGciOiJIUzI1NiIs..."
$ curl http://localhost:8080/me -H "Authorization: Bearer $TOKEN"
{"id":1,"email":"lev@example.com","created_at":"..."}

# /me без токена
$ curl -i http://localhost:8080/me
HTTP/1.1 401 Unauthorized
{"error":"missing authorization header"}

# /me с битым токеном
$ curl -i http://localhost:8080/me -H "Authorization: Bearer broken-token"
HTTP/1.1 401 Unauthorized
{"error":"invalid token"}

# /me с неправильным форматом
$ curl -i http://localhost:8080/me -H "Authorization: NotBearer abc"
HTTP/1.1 401 Unauthorized
{"error":"invalid authorization format"}
```

## Требования

### Зависимость

1. Установлена библиотека: `go get github.com/golang-jwt/jwt/v5`

### Auth

2. К `auth.Service` добавлены методы:
   - `GenerateToken(userID int64) (string, error)`
   - `ValidateToken(tokenStr string) (int64, error)`
3. Объявлена `ErrInvalidToken`
4. `GenerateToken` использует `jwt.NewWithClaims` с `MapClaims`:
   - `sub` = userID
   - `exp` = текущее время + 24 часа (Unix-секунды)
   - `iat` = текущее время (Unix-секунды)
5. Используется `jwt.SigningMethodHS256` и `s.secret` как ключ
6. `ValidateToken`:
   - Проверяет, что метод подписи — HMAC (через type assertion)
   - Возвращает `0, ErrInvalidToken` при любой ошибке парсинга
   - Извлекает `sub` через `claims["sub"].(float64)` и приводит к `int64`

### Storage

7. Добавлен метод `(r *UserRepository) GetByID(ctx, id) (*domain.User, error)`
8. Использует `QueryRowContext`
9. При `sql.ErrNoRows` → `ErrUserNotFound`

### API — login

10. Реализован handler `(h *Handler) login(w, r)`
11. Парсит JSON с `email` и `password`
12. Email нормализуется через `TrimSpace + ToLower`
13. Получает пользователя через `h.users.GetByEmail`
14. **При `ErrUserNotFound` → 401 Unauthorized с сообщением `"invalid credentials"`** (защита от user enumeration)
15. Проверяет пароль через `h.auth.CheckPassword(user.PasswordHash, input.Password)`
16. **При `ErrInvalidPassword` → 401 с тем же сообщением `"invalid credentials"`**
17. При успехе: генерирует токен, возвращает `200 OK` + `{"token": "..."}`
18. Эндпоинт `POST /login` зарегистрирован

### API — middleware

19. Создан файл `internal/api/middleware.go`
20. Объявлен **свой тип** `type contextKey string`
21. Объявлена константа `userIDKey contextKey = "user_id"`
22. Реализован метод `(h *Handler) authMiddleware(next http.Handler) http.Handler`
23. Middleware:
    - Получает заголовок `Authorization`
    - Если пустой → 401 «missing authorization header»
    - Парсит формат `Bearer <token>` через `strings.SplitN`
    - Если формат не «Bearer ...» → 401 «invalid authorization format»
    - Валидирует токен через `h.auth.ValidateToken`
    - При ошибке → 401 «invalid token»
    - **Кладёт `userID` в контекст** через `context.WithValue`
    - Вызывает `next.ServeHTTP(w, r.WithContext(ctx))`
24. Реализован helper `userIDFromContext(ctx) (int64, bool)`

### API — /me

25. Реализован handler `(h *Handler) me(w, r)`
26. Извлекает `userID` через `userIDFromContext(r.Context())`
27. Получает пользователя через `h.users.GetByID(r.Context(), userID)`
28. При успехе — возвращает 200 + JSON пользователя
29. При `ErrUserNotFound` → 404 (это странный случай — токен валиден, но пользователь удалён)
30. Любая другая ошибка → 500 + лог

### Регистрация роутов

31. `POST /login` — публичный
32. `GET /me` — **защищённый** через `mux.Handle("GET /me", h.authMiddleware(http.HandlerFunc(h.me)))`

### Поведение

33. Все 6 сценариев из примера работают
34. Эндпоинты модуля 2 продолжают работать
35. Логин после регистрации возвращает токен
36. Токен можно использовать для `/me`

## Шаги

1. Установи golang-jwt: `go get github.com/golang-jwt/jwt/v5`
2. Расширь `internal/auth/service.go` методами `GenerateToken`, `ValidateToken` и `ErrInvalidToken`
3. Добавь `GetByID` в `internal/storage/user.go`
4. Реализуй handler `login` в `internal/api/users.go`
5. Создай `internal/api/middleware.go` с `contextKey`, `userIDKey`, `authMiddleware`, `userIDFromContext`
6. Реализуй handler `me` в `internal/api/users.go`
7. Обнови `RegisterRoutes`:
   - Добавь `POST /login`
   - Добавь `GET /me` с middleware
8. Запусти, прогон всех 6 сценариев

## Эксперименты

1. **Открой свой токен на jwt.io:** скопируй JWT, открой [jwt.io](https://jwt.io), вставь. Увидишь header, payload, signature. **Заметь, что payload читаемый** — это не шифрование.

2. **Подмени payload вручную:**
   - Вставь свой токен в jwt.io
   - Поменяй `sub` на другое число
   - Скопируй новый токен (без секрета — подпись будет невалидна)
   - Пошли в `/me`
   - Получишь 401 «invalid token» — потому что подпись не сходится

3. **Сделай HMAC с правильным секретом:**
   - В jwt.io введи в поле «secret» твой `JWT_SECRET`
   - Поменяй `sub` на 999
   - Скопируй новый токен
   - Пошли в `/me`
   - Получишь 404 «user not found» (если пользователя 999 нет) или данные пользователя 999
   - **Это и есть смысл секрета**: он защищает от подделки

4. **Запросишь `/me` с истёкшим токеном:** временно поменяй `tokenLifetime` на `1 * time.Second`, перезапусти, залогинься, подожди 2 секунды, попробуй `/me`. Получишь 401. Это работа `exp`.

5. **Прислать токен в query string:** некоторые API принимают `?token=...`. У нас — нет, только заголовок. Это **более безопасно**, потому что URL-ы попадают в логи серверов.

6. **Залогиниться с несуществующим email:** убедись, что сообщение **точно такое же**, как при неправильном пароле. Это защита от user enumeration.

7. **Замерь, сколько занимает `GenerateToken`:** обычно меньше миллисекунды. JWT — быстрая операция в отличие от bcrypt.

8. **Сравни время `Login`:** логин = `GetByEmail` (быстро) + `CheckPassword` (медленно, ~100ms) + `GenerateToken` (быстро). Bottleneck — bcrypt.

9. **Попробуй передать токен в нижнем регистре:** `Authorization: bearer ...` (с маленькой буквы). Что произойдёт? У нас — 401, потому что мы проверяем `parts[0] != "Bearer"`. По спецификации **должно быть** case-insensitive. Можно поправить через `strings.EqualFold(parts[0], "Bearer")`.

10. **Запиши токен в файл и используй из shell-переменной:**
    ```bash
    TOKEN=$(curl -s -X POST http://localhost:8080/login -H "Content-Type: application/json" -d '{"email":"lev@example.com","password":"strong-password-123"}' | jq -r .token)
    curl http://localhost:8080/me -H "Authorization: Bearer $TOKEN"
    ```
    Это удобный шаблон для тестирования через `curl`.

## Что показать Claude

Покажи Claude свой `login` handler и спроси: «правильно ли я защищаюсь от user enumeration? Как ещё можно атаковать мой login?». Возможные ответы:

- **Timing attack** — даже если ответы одинаковые, время ответа разное: «не нашёл пользователя» = быстро (только SELECT), «нашёл, но пароль неверен» = медленно (SELECT + bcrypt). Атакующий может это измерить. **Защита** — всегда вызывать bcrypt, даже если пользователя нет (с фиктивным хешем).
- **Brute force** — атакующий перебирает пароли. **Защита** — rate limiting на эндпоинт `/login`. В нашем проекте пока без него.
- **Credential stuffing** — атакующий использует пары `email:password` из других утечек. **Защита** — bcrypt уже помогает (не быстро перебирать). Плюс капча, плюс rate limiting.

Запиши себе на будущее: **аутентификация — это бесконечная гонка**. Минимум, который ты должен делать всегда: bcrypt + rate limiting + одинаковые сообщения об ошибках. Дальше — по необходимости.
