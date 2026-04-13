# Модуль 1. Структура проекта — условие

## Задание

Создать **скелет** проекта с правильной структурой пакетов: `cmd/server/main.go`, `internal/api`, `internal/auth`, `internal/config`, `internal/domain`, `internal/storage`. Внутри — заглушки, которые компилируются и запускаются. Один работающий эндпоинт `GET /hello`.

## Что должно произойти

```bash
$ tree -L 3
.
├── go.mod
├── go.sum
├── README.md
├── .env.example
├── cmd
│   └── server
│       └── main.go
├── internal
│   ├── api
│   │   └── handler.go
│   ├── auth
│   │   └── service.go
│   ├── config
│   │   └── config.go
│   ├── domain
│   │   └── user.go
│   └── storage
│       └── user.go
└── migrations

$ DATABASE_URL=postgres://test JWT_SECRET=test go run ./cmd/server
2026/04/13 12:00:00 Сервер запускается на :8080

$ curl http://localhost:8080/hello
Hello, blog!
```

При отсутствии переменных окружения:

```bash
$ go run ./cmd/server
2026/04/13 12:00:00 config: DATABASE_URL is required
exit status 1
```

## Требования

### Структура

1. Создан корневой `go.mod` с **module path** (например, `github.com/yourname/blog` или `blog`)
2. Создана папка `cmd/server/` с файлом `main.go`
3. Создана папка `internal/` с подпапками `api`, `auth`, `config`, `domain`, `storage`
4. В каждой подпапке — **минимум один файл** с кодом
5. Имена пакетов совпадают с именами папок (`package api`, `package auth`, и т.д.)
6. Создана пустая папка `migrations/` (наполним в модуле 2)
7. Создан `.env.example` со списком переменных:
   ```
   DATABASE_URL=postgres://postgres:secret@localhost:5432/blog?sslmode=disable
   JWT_SECRET=change-me-in-production
   PORT=8080
   ```

### Код

8. В `internal/config/config.go`:
   - Тип `Config` со полями `DatabaseURL`, `JWTSecret`, `Port`
   - Функция `Load() (*Config, error)`
   - **`DATABASE_URL` обязателен** — ошибка, если пустой
   - **`JWT_SECRET` обязателен** — ошибка, если пустой
   - **`PORT`** — по умолчанию `"8080"`, если пусто

9. В `internal/domain/user.go`:
   - Тип `User` с полями `ID int64`, `Email string`, `CreatedAt time.Time`
   - **JSON-теги** на всех полях
   - **Никакого `PasswordHash` в JSON-выводе** — будем добавлять в модуле 2

10. В `internal/storage/user.go`:
    - Тип `UserRepository` с полем `db *sql.DB`
    - Конструктор `NewUserRepository(db *sql.DB) *UserRepository`
    - Метод-заглушка (например, `Ping() error` или комментарий «методы появятся в модуле 2»)

11. В `internal/auth/service.go`:
    - Тип `Service` с полем `secret string`
    - Конструктор `NewService(secret string) *Service`
    - Метод-заглушка

12. В `internal/api/handler.go`:
    - Тип `Handler` с полями `users *storage.UserRepository`, `auth *auth.Service`
    - Конструктор `NewHandler(users, auth) *Handler`
    - Метод `RegisterRoutes(mux *http.ServeMux)` — регистрирует `GET /hello`
    - Метод `hello(w, r)` — отвечает «Hello, blog!»

13. В `cmd/server/main.go`:
    - Импорт всех `internal/...` пакетов
    - `config.Load()` с `log.Fatalf` при ошибке
    - Создание зависимостей через конструкторы (с `nil` вместо `*sql.DB` пока)
    - Регистрация роутов
    - `http.ListenAndServe`

### Запуск

14. Команда `go run ./cmd/server` запускает сервер
15. **`curl http://localhost:8080/hello`** возвращает `Hello, blog!`
16. При отсутствии `DATABASE_URL` или `JWT_SECRET` — выход с ошибкой
17. **Все импорты between пакетами через полный путь** (`import "blog/internal/api"`), не через относительные

## Шаги

1. Создай папку `007-blog-platform/`:
   ```bash
   mkdir 007-blog-platform && cd 007-blog-platform
   ```

2. Инициализируй модуль (выбери имя, например `blog`):
   ```bash
   go mod init blog
   ```

3. Создай структуру папок:
   ```bash
   mkdir -p cmd/server internal/api internal/auth internal/config internal/domain internal/storage migrations
   ```

4. Создай `.env.example`

5. Создай `internal/config/config.go` с типом `Config` и `Load()`

6. Создай `internal/domain/user.go` с типом `User`

7. Создай `internal/storage/user.go` с типом `UserRepository` (заглушка)

8. Создай `internal/auth/service.go` с типом `Service` (заглушка)

9. Создай `internal/api/handler.go` с типом `Handler`, методами `RegisterRoutes` и `hello`

10. Создай `cmd/server/main.go` с инициализацией и запуском

11. Запусти:
    ```bash
    DATABASE_URL=postgres://test JWT_SECRET=test go run ./cmd/server
    ```

12. В другом терминале:
    ```bash
    curl http://localhost:8080/hello
    ```

13. Проверь поведение без переменных

14. **Закоммить с осмысленным сообщением**, например `init: project skeleton`

## Эксперименты

1. **Попробуй вызвать приватную функцию из другого пакета:** добавь в `internal/auth/service.go` функцию `func generateSecret() string { ... }` (с маленькой буквы). Попробуй вызвать её из `cmd/server/main.go`:
   ```go
   auth.generateSecret()  // КОМПИЛЯТОР НЕ ДАСТ
   ```
   Получишь `cannot refer to unexported name auth.generateSecret`. Сделай большую букву — заработает. Это правило видимости в действии.

2. **Создай циклическую зависимость:** в `internal/storage/user.go` импортируй `internal/api`. В `internal/api/handler.go` импортируй `internal/storage` (что у тебя уже есть). Что произойдёт?
   ```
   import cycle not allowed
   ```
   Go запрещает циклические зависимости на уровне компилятора. Это **очень полезное** ограничение — оно заставляет проектировать чисто.

3. **Попробуй запустить через `go run main.go`** (без `./cmd/server`). Что произойдёт? Ничего — этого файла больше нет в текущей директории. Используй `go run ./cmd/server`.

4. **Попробуй вызвать `cmd/server/main.go` извне:** `go build -o blog ./cmd/server` создаёт исполняемый файл `blog`. Запусти его:
   ```bash
   DATABASE_URL=test JWT_SECRET=test ./blog
   ```
   Это и есть **сборка** проекта — то, что в продакшене запускается на сервере.

5. **Попробуй импортировать `internal/...` из вне модуля:** создай рядом отдельную папку `external/` с собственным `go.mod` и попробуй импортировать `blog/internal/api`. Что произойдёт?
   ```
   use of internal package not allowed
   ```
   Это работа `internal/` — она защищает от внешних импортов.

6. **Запусти `go vet ./...`** — статический анализатор. Должен пройти без ошибок. Если что-то не так — поправь.

7. **Запусти `go fmt ./...`** — автоматический форматёр. Если что-то изменилось, посмотри. Привычка хорошая для всех проектов.

8. **Используй `go list ./...`** — посмотришь все пакеты проекта. Должно быть что-то вроде:
   ```
   blog/cmd/server
   blog/internal/api
   blog/internal/auth
   blog/internal/config
   blog/internal/domain
   blog/internal/storage
   ```

## Что показать Claude

Покажи Claude свою структуру через `tree` и спроси: «правильно ли я выделил слои? Что бы я добавил, если бы у меня была сложная бизнес-логика?». Возможные ответы:

- **`internal/service/`** — бизнес-логика между handler и repository (Variant B из теории)
- **`internal/middleware/`** — отдельный пакет для middleware (для нашего проекта останется в `internal/api/`)
- **`internal/db/`** или **`internal/postgres/`** — отдельный пакет для подключения и пула, чтобы `storage` не делал `sql.Open`
- **`internal/handlers/users/`**, **`internal/handlers/posts/`** — разделение handler-ов по доменам, если их много

Запиши себе на будущее: **архитектура должна расти с проектом**. Сначала простая (5-6 пакетов), потом постепенно усложняется по мере роста. **Не делай сложно с самого начала** — это самая частая ошибка новичков, прочитавших про DDD.
