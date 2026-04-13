# Модуль 2. Users и регистрация — чек-лист готовности

## Поведенческие маркеры

### Миграция и БД

- [ ] Создан файл `migrations/001_create_users.sql`
- [ ] Таблица `users` имеет 4 колонки: `id BIGSERIAL`, `email TEXT NOT NULL UNIQUE`, `password_hash TEXT NOT NULL`, `created_at TIMESTAMPTZ`
- [ ] Создана БД `blog`
- [ ] Миграция применена

### Зависимости

- [ ] Установлен `golang.org/x/crypto/bcrypt`
- [ ] Установлен `github.com/jackc/pgx/v5/stdlib` (если ещё не было)
- [ ] Импорт драйвера через `_ "github.com/jackc/pgx/v5/stdlib"` в main

### Domain

- [ ] Тип `User` имеет поле `PasswordHash string` с тегом `json:"-"`

### Storage

- [ ] Объявлены `ErrUserNotFound` и `ErrEmailExists`
- [ ] Реализован `Create(ctx, email, passwordHash) (*domain.User, error)`
- [ ] `Create` использует `INSERT ... RETURNING ...`
- [ ] `Create` использует `QueryRowContext` с контекстом
- [ ] Реализована функция `isUniqueViolation(err) bool` через `errors.As + pgconn.PgError`
- [ ] При `unique_violation (23505)` `Create` возвращает `ErrEmailExists`
- [ ] Реализован `GetByEmail(ctx, email) (*domain.User, error)`
- [ ] При `sql.ErrNoRows` `GetByEmail` возвращает `ErrUserNotFound`
- [ ] Все ошибки обёрнуты через `%w`

### Auth

- [ ] Реализован `(s *Service) HashPassword(password) (string, error)`
- [ ] Реализован `(s *Service) CheckPassword(hash, password) error`
- [ ] Объявлена `ErrInvalidPassword`
- [ ] `CheckPassword` ловит `bcrypt.ErrMismatchedHashAndPassword` и возвращает `ErrInvalidPassword`

### API

- [ ] Реализован handler `register` в `internal/api/users.go`
- [ ] Парсит JSON через `json.NewDecoder(r.Body).Decode`
- [ ] Email нормализуется через `TrimSpace + ToLower`
- [ ] Email валидируется через регулярку
- [ ] Пароль валидируется (минимум 8 символов)
- [ ] Хеширует пароль через `h.auth.HashPassword`
- [ ] Создаёт пользователя через `h.users.Create(r.Context(), ...)`
- [ ] При `ErrEmailExists` → 409 Conflict
- [ ] При других ошибках → лог + 500
- [ ] При успехе → 201 Created + JSON
- [ ] **`password_hash` НЕ появляется в ответе**
- [ ] Реализованы helpers `writeJSON` и `writeError`
- [ ] Эндпоинт `POST /register` зарегистрирован в `RegisterRoutes`

### Main

- [ ] `sql.Open("pgx", cfg.DatabaseURL)` с обработкой ошибки
- [ ] `defer db.Close()`
- [ ] Настройка пула (`SetMaxOpenConns`, etc.)
- [ ] `db.Ping()` сразу после Open
- [ ] `userRepo` создаётся с **реальным** `db`, не `nil`
- [ ] Лог «Connected to PostgreSQL»

### Тесты сценариев

- [ ] Успешная регистрация → 201 + JSON без пароля
- [ ] Дубликат email → 409
- [ ] Невалидный email → 400
- [ ] Слабый пароль → 400
- [ ] Невалидный JSON → 400
- [ ] Хеш в БД виден через `psql` (начинается с `$2a$10$`)
- [ ] `GET /hello` из модуля 1 продолжает работать
- [ ] Код закоммичен

## Концептуальные маркеры

- [ ] Знаю, **почему** нельзя хранить пароли в plain text
- [ ] Понимаю, что такое **соль** и зачем она
- [ ] Знаю, **почему** bcrypt намеренно медленный
- [ ] Перечисляю плохие варианты хеш-функций (MD5, SHA1, голый SHA256)
- [ ] Понимаю формат хеша bcrypt (`$2a$10$...`)
- [ ] Знаю два главных метода bcrypt в Go
- [ ] Могу объяснить, **зачем** `json:"-"` на `PasswordHash`
- [ ] Понимаю, что такое **sentinel error** и как ловить через `errors.Is`/`errors.As`
- [ ] Знаю, что **`23505`** — это `unique_violation` в PostgreSQL
- [ ] Понимаю, что **доменные ошибки** должны жить отдельно от драйверных
- [ ] Знаю, **зачем** нормализовать email перед сохранением
- [ ] Могу объяснить, **почему** «email уже существует» — это **409 Conflict**, а не 400 или 500

## Маркеры экспериментов

- [ ] **Прочитал хеш через `psql`**, увидел формат bcrypt
- [ ] Захешировал один пароль дважды, увидел разные хеши (соль работает)
- [ ] Замерил время `HashPassword` (~50-200ms)
- [ ] Прислал невалидный JSON, увидел 400
- [ ] Зарегистрировал email в верхнем регистре, проверил нормализацию
- [ ] Зарегистрировал с пробелами вокруг email, проверил `TrimSpace`
- [ ] (Опционально) Попробовал очень длинный пароль, увидел поведение bcrypt
- [ ] Сделал `SELECT * FROM users` через `psql` — увидел структуру

## Самопроверка

Не подсматривая, ответь:

1. Какие два метода bcrypt я использую?
2. Какой код PostgreSQL означает «дубликат UNIQUE»?
3. Как получить из ошибки драйвера тип `*pgconn.PgError`?
4. Какой тег скрывает поле от JSON?
5. Какой статус возвращать при «email уже занят»?
6. Зачем нормализовать email через `ToLower`?

## Подготовка к модулю 3

В модуле 3 ты добавишь **аутентификацию по токену**:

- POST `/login` — выдача JWT после проверки email + пароля
- Метод `auth.GenerateToken(userID)` через `github.com/golang-jwt/jwt/v5`
- Метод `auth.ValidateToken(tokenStr) (userID int64, err error)`
- **Middleware** для защищённых эндпоинтов
- GET `/me` — возвращает информацию о текущем пользователе (требует токен)
- Передача `user_id` через `r.Context()` от middleware в handler

К концу модуля 3 у тебя будет **рабочая аутентификация**: пользователь логинится, получает токен, и шлёт его в `Authorization: Bearer <token>` для защищённых запросов.
