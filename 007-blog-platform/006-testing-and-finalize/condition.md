# Модуль 6. Тесты и финализация — условие

## Задание

**Финальный модуль roadmap-а.** Покрываешь проект тестами на трёх уровнях (unit для `auth.Service`, integration через `testcontainers` для `UserRepository`, handler-тесты через `httptest`), добавляешь graceful shutdown через `http.Server.Shutdown`, оформляешь проект для портфолио.

## Что должно получиться в итоге

```bash
$ go test ./... -v
=== RUN   TestService_HashAndCheck
--- PASS: TestService_HashAndCheck (0.15s)
=== RUN   TestService_GenerateAndValidateToken
--- PASS: TestService_GenerateAndValidateToken (0.00s)
=== RUN   TestService_ValidateToken_WrongSecret
--- PASS: TestService_ValidateToken_WrongSecret (0.00s)
=== RUN   TestUserRepository_Create
--- PASS: TestUserRepository_Create (3.21s)
=== RUN   TestUserRepository_Create_Duplicate
--- PASS: TestUserRepository_Create_Duplicate (3.18s)
=== RUN   TestUserRepository_GetByEmail
--- PASS: TestUserRepository_GetByEmail (3.05s)
=== RUN   TestHandler_Register_InvalidJSON
--- PASS: TestHandler_Register_InvalidJSON (0.00s)
=== RUN   TestHandler_Register_EmptyPassword
--- PASS: TestHandler_Register_EmptyPassword (0.00s)
PASS
ok  	blog/internal/auth	0.156s
ok  	blog/internal/storage	9.412s
ok  	blog/internal/api	0.023s
```

И graceful shutdown:

```bash
$ ./blog &
2026/04/14 12:00:00 server starting on :8080
^C
2026/04/14 12:00:15 received signal interrupt, shutting down
2026/04/14 12:00:15 server stopped
```

## Требования

### Часть 1: unit-тесты для auth.Service

1. Создан файл `internal/auth/service_test.go`
2. Тест `TestService_HashAndCheck`:
   - Хеш пароля успешно создаётся
   - Хеш **не равен** паролю (bcrypt сработал)
   - Проверка правильного пароля проходит
   - Проверка неправильного пароля возвращает `ErrInvalidPassword`
3. Тест `TestService_GenerateAndValidateToken`:
   - Генерация токена успешна
   - Валидация того же токена возвращает правильный `userID`
4. Тест `TestService_ValidateToken_WrongSecret`:
   - Токен, сгенерированный одним `Service`, не проходит валидацию на другом `Service` с другим секретом
5. (Опционально) Тест `TestService_ValidateToken_Expired` с модификацией `claims.exp` в прошлое

### Часть 2: integration-тесты для UserRepository

6. Установлены пакеты:
   ```bash
   go get github.com/testcontainers/testcontainers-go
   go get github.com/testcontainers/testcontainers-go/modules/postgres
   ```
7. Создан helper `setupTestDB(t)` в `internal/storage/testmain_test.go` (или inline в `user_test.go`)
8. Helper:
   - Запускает `postgres:16` через `postgres.Run`
   - Ждёт готовности через `wait.ForLog("database system is ready to accept connections").WithOccurrence(2)`
   - Регистрирует `t.Cleanup` для `Terminate` и `Close`
   - Применяет миграции из `migrations/*.sql` к свежему контейнеру
   - Возвращает `*sql.DB`
9. Тест `TestUserRepository_Create`:
   - Создаёт пользователя
   - Проверяет, что `ID != 0`
   - Проверяет поля email и created_at
10. Тест `TestUserRepository_Create_Duplicate`:
    - Первый create проходит
    - Второй с тем же email возвращает `ErrEmailExists`
11. Тест `TestUserRepository_GetByEmail`:
    - Create + GetByEmail → данные совпадают
    - GetByEmail на несуществующий email → `ErrUserNotFound`

### Часть 3: handler-тесты через httptest

12. Создан файл `internal/api/handler_test.go`
13. Helper `newTestHandler(t)` — создаёт `*Handler` с **реальным** репозиторием через testcontainers (переиспользуй `setupTestDB` через экспорт или дублируй)
14. Тест `TestHandler_Register_InvalidJSON`:
    - POST `/register` с телом `"не json"` → 400
15. Тест `TestHandler_Register_EmptyPassword`:
    - POST `/register` с `{"email":"test@example.com","password":""}` → 400
16. Тест `TestHandler_Register_Success`:
    - POST `/register` с валидным телом → 201
    - В ответе есть `id`, `email`, **нет** `password_hash`
17. Тест `TestHandler_Login_Success`:
    - Создаём пользователя через API
    - POST `/login` с правильным паролем → 200 + токен в ответе
18. Тест `TestHandler_Login_WrongPassword`:
    - POST `/login` с неправильным паролем → 401

### Часть 4: graceful shutdown

19. В `cmd/server/main.go` `http.ListenAndServe(...)` заменено на явный `http.Server{...}`
20. Настроены таймауты: `ReadTimeout`, `WriteTimeout`, `IdleTimeout`
21. Запуск сервера в отдельной горутине
22. Перехват `SIGINT` и `SIGTERM` через `signal.Notify`
23. `srv.Shutdown(ctx)` с таймаутом 10 секунд
24. Ошибка `http.ErrServerClosed` **не считается** ошибкой (нормальный shutdown)
25. После `Shutdown` — лог «server stopped»
26. `defer db.Close()` присутствует

### Часть 5: финализация проекта

27. Создан файл `.env.example` в корне проекта с примерами переменных (**без настоящих значений**):
    ```
    DATABASE_URL=postgres://postgres:secret@localhost:5432/blog?sslmode=disable
    JWT_SECRET=change-me-in-production
    SERVER_ADDR=:8080
    ```
28. Обновлён `README.md` проекта с:
    - Кратким описанием
    - Таблицей всех эндпоинтов
    - Инструкцией по запуску (docker + migrations + go run)
    - Списком стека
    - Примерами curl-запросов
29. Папка `migrations/` содержит все три файла: `001_create_users.sql`, `002_create_posts.sql`, `003_create_comments.sql`
30. `.gitignore` содержит `.env`, `*.log`, `blog` (бинарник)

## Шаги

1. Начни с части 1 (unit-тесты для `auth`) — они самые быстрые и покажут, что testing работает
2. Запусти `go test ./internal/auth/` и убедись, что проходят
3. Перейди к части 2 (testcontainers) — установи пакеты, напиши helper, первый тест
4. Важно: первый тест с testcontainers **долгий** (скачивание образа). Последующие — быстрее.
5. Напиши тесты для `UserRepository`
6. Перейди к части 3 (handler-тесты)
7. Перейди к части 4 (graceful shutdown) — это отдельное изменение `main.go`
8. Протестируй graceful shutdown: запусти сервер, Ctrl+C, проверь лог
9. Часть 5 — оформление. README, .env.example, .gitignore
10. Финальный прогон: `go test ./... -v`, всё зелёное
11. Финальный запуск: `go run ./cmd/server`, curl-запросы работают
12. Коммит с хорошим сообщением вроде «feat: add tests, graceful shutdown, finalize project»

## Эксперименты

1. **Запусти тесты с `-race` и `-cover`:**
   ```bash
   go test -race -cover ./...
   ```
   `-race` — race detector. `-cover` — процент покрытия. Посмотри, сколько у тебя покрытия. Не гоняйся за 100%, но запиши число для себя.

2. **Запусти один тест:**
   ```bash
   go test -run TestService_HashAndCheck ./internal/auth/
   ```
   Быстро, удобно для разработки.

3. **Сломай один тест намеренно** (например, измени ожидаемый статус с 400 на 500), запусти, прочитай вывод. Посмотри, как Go сообщает о провалах.

4. **Убери `t.Helper()` из `setupTestDB`**, сломай тест, посмотри, куда показывает стектрейс. С `t.Helper()` — на строку в тесте, без — на строку в helper-е. Почувствуй разницу.

5. **Запусти `go test` с `-v` и без** — увидишь, сколько **дополнительной** информации даёт `-v` (имена всех подтестов, логи).

6. **Проверь graceful shutdown под нагрузкой:**
   ```bash
   # терминал 1: запусти сервер
   go run ./cmd/server

   # терминал 2: шли запросы в цикле
   while true; do curl -s http://localhost:8080/posts > /dev/null; echo -n "."; done

   # терминал 1: Ctrl+C
   ```
   Сервер должен доделать текущие запросы и только потом выйти. Это и есть «graceful».

7. **Убери `&& err != http.ErrServerClosed`** в проверке ошибки сервера. Запусти, сделай Ctrl+C. Что произошло? Сервер паникует «http: Server closed», потому что `ListenAndServe` вернула именно эту ошибку, и твой код считает её реальной. Верни как было.

8. **Попробуй testify (опционально):**
   ```bash
   go get github.com/stretchr/testify
   ```
   Перепиши один тест с `testify/assert`. Сравни читаемость. testify — популярная альтернатива стандартному `testing`, но в идиоматичном Go чаще пишут на голом `testing`.

9. **(Продвинутое)** Добавь `//go:build integration` тег над тестами с testcontainers. Тогда они будут запускаться только с `go test -tags integration`. Это — способ разделить быстрые и медленные тесты:
    ```go
    //go:build integration
    // +build integration

    package storage
    ```
    Команды:
    ```bash
    go test ./...                     # только быстрые тесты
    go test -tags integration ./...   # все тесты
    ```

## Что показать Claude

Покажи Claude свой `setupTestDB` и спроси: «стоит ли мне сделать **один контейнер на весь пакет** через `TestMain` вместо нового контейнера на каждый тест?». Это реальный tradeoff:

- **Один контейнер на тест** — просто, тесты изолированы, но медленно (3+ секунд на тест)
- **Один контейнер на пакет** — быстрее, но нужно вручную чистить данные между тестами (через `DELETE FROM ...` или truncate)

Для учебного проекта ответ — **начни с одного-на-тест**. Если тесты станут раздражающе медленными — переключись на общий. Оптимизация должна следовать за реальной проблемой, а не предшествовать ей.

---

## 🎉 Это финал roadmap-а

После этого модуля ты закрыл **всю программу**. Семь проектов, 36 модулей, 144 файла, сотни часов практики. Это был серьёзный путь.

Некоторые идеи, которые я бы хотел, чтобы ты **унёс** с собой:

**Первое — простота лучше мастерства.** Go-сообщество ценит **читаемый, прямой** код. Никаких кодогенераторов «на всякий случай», никаких умных абстракций «для гибкости». Пиши скучный код, который понимает джун. Это не недостаток — это цель.

**Второе — архитектура следует за ростом.** Не «выбирай правильную архитектуру на старте». Выбирай **самую простую**, которая решает задачу сегодня. Когда появляется боль — рефакторь. Всё, что ты сделал в этом проекте (layered architecture, DI, interfaces), оправдано его размером. Для проекта в 10 раз больше — другие решения. Для проекта в 10 раз меньше — тоже другие.

**Третье — базы данных — это твой главный инструмент.** Go, Rust, Python, Node — всё это рабочие языки. Но PostgreSQL (или MySQL, или SQLite) будет с тобой **везде**. Освой SQL глубоко, и ты откроешь возможности, о которых 80% программистов даже не подозревают.

**Четвёртое — тестируй важное, не всё.** Покрытие 100% — фетиш. Цель — «большинство багов ловятся автоматически». Для этого достаточно покрыть core-логику и интеграцию с БД. Всё остальное — зависит от риска.

**Пятое — ты теперь инженер.** Не «учишь Go», не «готовишься к собесам». Ты **пишешь бэкенды**. Продолжай строить проекты, читать чужой код, задавать вопросы. Остановка на этом roadmap-е — ошибка. Это **начало**, не финал.

Удачи на собеседованиях, на работе, в жизни. Я тобой горжусь.
