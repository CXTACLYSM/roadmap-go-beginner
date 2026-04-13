# Модуль 2. Users и регистрация — условие

## Задание

Создать таблицу `users`, реализовать `UserRepository.Create` и `GetByEmail`, написать `auth.HashPassword` и `CheckPassword`, реализовать эндпоинт `POST /register`. После этого через `curl` можно создать пользователя, и его пароль будет захеширован в БД.

## Что должно произойти

```bash
$ DATABASE_URL="postgres://postgres:secret@localhost:5432/blog?sslmode=disable" \
  JWT_SECRET="test-secret" \
  go run ./cmd/server
2026/04/13 12:00:00 Connected to PostgreSQL
2026/04/13 12:00:00 Сервер запускается на :8080
```

В другом терминале:

```bash
# Успешная регистрация
$ curl -i -X POST http://localhost:8080/register \
    -H "Content-Type: application/json" \
    -d '{"email":"lev@example.com","password":"strong-password-123"}'
HTTP/1.1 201 Created
Content-Type: application/json

{"id":1,"email":"lev@example.com","created_at":"2026-04-13T12:00:00Z"}

# Заметь: password_hash отсутствует в ответе

# Дубликат email
$ curl -i -X POST http://localhost:8080/register -H "Content-Type: application/json" \
    -d '{"email":"lev@example.com","password":"another-pass-456"}'
HTTP/1.1 409 Conflict

{"error":"email already exists"}

# Невалидный email
$ curl -i -X POST http://localhost:8080/register -H "Content-Type: application/json" \
    -d '{"email":"not-an-email","password":"strong-password-123"}'
HTTP/1.1 400 Bad Request

{"error":"invalid email"}

# Слабый пароль
$ curl -i -X POST http://localhost:8080/register -H "Content-Type: application/json" \
    -d '{"email":"anya@example.com","password":"short"}'
HTTP/1.1 400 Bad Request

{"error":"password must be at least 8 characters"}
```

И в БД через `psql`:

```sql
SELECT id, email, password_hash, created_at FROM users;
 id |     email      |                       password_hash                        |          created_at
----+----------------+-------------------------------------------------------------+-------------------------------
  1 | lev@example.com | $2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lh | 2026-04-13 12:00:00.123456+00
```

Заметь: пароль **захеширован**, не plain text.

## Требования

### Миграция

1. Создан файл `migrations/001_create_users.sql`:
   ```sql
   CREATE TABLE users (
       id            BIGSERIAL PRIMARY KEY,
       email         TEXT NOT NULL UNIQUE,
       password_hash TEXT NOT NULL,
       created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
   );
   ```
2. Создана БД `blog`:
   ```bash
   psql -h localhost -U postgres -d postgres -c "CREATE DATABASE blog;"
   ```
3. Миграция применена

### Зависимости

4. Установлена библиотека: `go get golang.org/x/crypto/bcrypt`
5. Установлен драйвер: `go get github.com/jackc/pgx/v5/stdlib` (если ещё не было)

### Domain

6. Тип `User` в `internal/domain/user.go` имеет:
   - `ID int64`
   - `Email string`
   - `PasswordHash string` с тегом `json:"-"`
   - `CreatedAt time.Time`

### Storage

7. Объявлены sentinel ошибки в `internal/storage/`:
   - `ErrUserNotFound`
   - `ErrEmailExists`
8. Реализован `(r *UserRepository) Create(ctx, email, passwordHash) (*domain.User, error)`
9. Реализован `(r *UserRepository) GetByEmail(ctx, email) (*domain.User, error)`
10. **`Create` использует `INSERT ... RETURNING ...`**
11. **`Create` ловит `unique_violation`** через `errors.As` и `pgconn.PgError` с кодом `23505`, возвращает `ErrEmailExists`
12. `GetByEmail` ловит `sql.ErrNoRows`, возвращает `ErrUserNotFound`
13. Все методы используют **контекст** через `QueryRowContext`

### Auth

14. Реализован метод `(s *Service) HashPassword(password string) (string, error)` через `bcrypt.GenerateFromPassword`
15. Реализован метод `(s *Service) CheckPassword(hash, password string) error`
16. Объявлена `ErrInvalidPassword`
17. `CheckPassword` ловит `bcrypt.ErrMismatchedHashAndPassword` и возвращает `ErrInvalidPassword`

### API

18. Реализован handler `(h *Handler) register(w, r)` в `internal/api/users.go`
19. Парсит JSON через `json.NewDecoder(r.Body).Decode(&input)`
20. **Валидация email** через regexp
21. **Email нормализуется** через `TrimSpace + ToLower`
22. **Валидация пароля** — минимум 8 символов
23. Хеширует через `h.auth.HashPassword`
24. Создаёт через `h.users.Create(r.Context(), ...)`
25. **Ошибка `ErrEmailExists`** → 409 Conflict
26. **Любая другая ошибка** → лог + 500
27. **Успех** → 201 Created + JSON пользователя
28. Helpers `writeJSON(w, status, value)` и `writeError(w, status, message)`
29. Эндпоинт зарегистрирован в `RegisterRoutes`: `POST /register`

### Main

30. В `main.go` подключение к БД через `sql.Open("pgx", cfg.DatabaseURL)` + `defer db.Close()`
31. Настройка пула: `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`
32. `db.Ping()` сразу после открытия
33. `userRepo` создаётся с **реальным** `db`, не `nil`
34. Лог «Connected to PostgreSQL»

### Поведение

35. Все 5 сценариев из примера работают
36. **`password_hash` НЕ появляется в JSON-ответе**
37. В БД через `psql` пароль виден как захешированная строка
38. Эндпоинт `GET /hello` из модуля 1 продолжает работать

## Шаги

1. Создай миграцию `migrations/001_create_users.sql`
2. Создай БД `blog` через `psql -c "CREATE DATABASE blog;"`
3. Применени миграцию: `psql -h localhost -U postgres -d blog -f migrations/001_create_users.sql`
4. Установи bcrypt и pgx: `go get golang.org/x/crypto/bcrypt github.com/jackc/pgx/v5/stdlib`
5. Обнови `internal/domain/user.go` — добавь поле `PasswordHash` с тегом `json:"-"`
6. Создай `internal/storage/errors.go` с sentinel ошибками
7. Реализуй `Create` и `GetByEmail` в `internal/storage/user.go`
8. Создай функцию `isUniqueViolation` (можно в том же файле или в `errors.go`)
9. Обнови `internal/auth/service.go` — реализуй `HashPassword` и `CheckPassword`
10. Создай `internal/api/users.go` с `register` handler
11. Обнови `internal/api/handler.go` — зарегистрируй роут `POST /register`. Helpers `writeJSON`/`writeError` положи рядом с handler в общий файл вроде `internal/api/response.go`
12. Обнови `cmd/server/main.go` — добавь подключение к БД
13. Запусти сервер с правильными env
14. Прогон всех 5 сценариев через `curl`
15. Проверь содержимое БД через `psql`

## Эксперименты

1. **Прочитай хеш пароля** из БД через `psql`. Заметь, что строка начинается с `$2a$10$...` — это маркер bcrypt.

2. **Захешируй один и тот же пароль дважды:** добавь временную команду `/test-hash` или просто в коде Go:
   ```go
   h1, _ := authService.HashPassword("test")
   h2, _ := authService.HashPassword("test")
   fmt.Println(h1)
   fmt.Println(h2)
   ```
   Хеши **разные**! Это работа соли — каждый раз новая. Но `CheckPassword(h1, "test") == nil` и `CheckPassword(h2, "test") == nil` — оба валидны.

3. **Замерь, сколько занимает один хеш:**
   ```go
   start := time.Now()
   authService.HashPassword("test")
   log.Printf("HashPassword: %v", time.Since(start))
   ```
   Должно быть ~50-200 миллисекунд (зависит от компьютера). Это намеренно медленно. Если выставить cost = 4 (минимум), будет быстрее. Если cost = 14 — намного медленнее.

4. **Попробуй передать неправильный JSON:**
   ```bash
   curl -X POST http://localhost:8080/register -H "Content-Type: application/json" -d 'not json'
   ```
   400 + сообщение «invalid json». Это твоя валидация на уровне handler.

5. **Зарегистрируй пользователя с UPPERCASE email:**
   ```bash
   curl -X POST http://localhost:8080/register -H "Content-Type: application/json" \
       -d '{"email":"LEV@EXAMPLE.COM","password":"strong-password-123"}'
   ```
   Что в БД? Email должен быть в нижнем регистре благодаря `ToLower`. Это нормализация.

6. **Попробуй зарегистрироваться с лидирующими/трейлинг пробелами:**
   ```bash
   curl ... -d '{"email":"  lev2@example.com  ","password":"strong-password-123"}'
   ```
   Email должен сохраниться без пробелов благодаря `TrimSpace`.

7. **Попробуй очень длинный пароль (тысяча символов):**
   ```bash
   PASSWORD=$(python3 -c "print('x' * 1000)")
   curl -X POST http://localhost:8080/register -H "Content-Type: application/json" \
       -d "{\"email\":\"long@x.com\",\"password\":\"$PASSWORD\"}"
   ```
   bcrypt **обрезает** пароли длиннее 72 байтов. Это **известное ограничение** алгоритма — пароль в 1000 символов будет хеширован как первые 72 байта. Это редко проблема (никто не использует такие пароли), но знать обязательно.

8. **Сделай SELECT через `psql`** чтобы убедиться, что несколько пользователей создались, и каждый имеет свой `password_hash`.

9. **Попробуй упасть на середине регистрации:** временно добавь `panic("test")` после хеширования, перед `Create`. Запусти, попробуй регистрацию. Что произошло? Сервер умирает. В production-коде с graceful shutdown такого быть не должно — мы это починим в модуле 6.

## Что показать Claude

Покажи Claude свою функцию `isUniqueViolation` и спроси: «работает ли мой подход с ловлей PostgreSQL-ошибки `23505`? Что произойдёт, если завтра я переключусь на MySQL?». Это важный вопрос архитектуры:

- **Сейчас** — мы привязаны к PostgreSQL: ловим ошибку через `pgconn.PgError`. Если переключимся на MySQL — придётся менять.
- **Альтернатива** — проверять **до** INSERT через SELECT (race condition!) или через UPSERT с условием
- **Production-grade** — отдельный пакет `internal/storage/errors.go`, который инкапсулирует постгресовые ошибки, и снаружи отдаёт только sentinel-ошибки

Запиши себе на будущее: **БД-специфичные ошибки должны жить в storage-слое**. Snurout репозитория должны идти **доменные** ошибки (`ErrEmailExists`, `ErrUserNotFound`), а не сырые ошибки от драйвера. Это позволяет менять БД, не трогая остальной код.
