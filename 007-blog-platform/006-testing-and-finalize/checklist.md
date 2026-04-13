# Модуль 6. Тесты и финализация — чек-лист готовности

## Поведенческие маркеры: unit-тесты

- [ ] Создан файл `internal/auth/service_test.go`
- [ ] Тест `TestService_HashAndCheck` — хеш + правильный пароль + неправильный пароль
- [ ] Тест `TestService_GenerateAndValidateToken` — round-trip токена
- [ ] Тест `TestService_ValidateToken_WrongSecret` — токен с чужим секретом отклоняется
- [ ] `go test ./internal/auth/` проходит

## Поведенческие маркеры: integration-тесты

- [ ] Установлены `testcontainers-go` и модуль `postgres`
- [ ] Реализован `setupTestDB(t *testing.T) *sql.DB`
- [ ] Helper помечен как `t.Helper()`
- [ ] `t.Cleanup(Terminate)` и `t.Cleanup(db.Close)` присутствуют
- [ ] `wait.ForLog("...").WithOccurrence(2)` для правильного ожидания PostgreSQL
- [ ] Миграции применяются после подключения к БД
- [ ] Тест `TestUserRepository_Create` проходит
- [ ] Тест `TestUserRepository_Create_Duplicate` возвращает `ErrEmailExists`
- [ ] Тест `TestUserRepository_GetByEmail` — Create + Get, потом несуществующий → `ErrUserNotFound`
- [ ] `go test ./internal/storage/` проходит

## Поведенческие маркеры: handler-тесты

- [ ] Создан `internal/api/handler_test.go`
- [ ] Helper `newTestHandler(t)` использует реальный репозиторий через testcontainers
- [ ] Тест `TestHandler_Register_InvalidJSON` → 400
- [ ] Тест `TestHandler_Register_EmptyPassword` → 400
- [ ] Тест `TestHandler_Register_Success` → 201, в ответе нет `password_hash`
- [ ] Тест `TestHandler_Login_Success` → 200 + токен
- [ ] Тест `TestHandler_Login_WrongPassword` → 401
- [ ] `go test ./internal/api/` проходит

## Поведенческие маркеры: graceful shutdown

- [ ] `main.go` использует явный `http.Server{...}` вместо `http.ListenAndServe(...)`
- [ ] Настроены `ReadTimeout`, `WriteTimeout`, `IdleTimeout`
- [ ] `srv.ListenAndServe()` запускается **в горутине**
- [ ] Ошибка `http.ErrServerClosed` игнорируется в проверке
- [ ] Перехват `os.Interrupt` и `syscall.SIGTERM` через `signal.Notify`
- [ ] `context.WithTimeout(ctx, 10*time.Second)` для shutdown
- [ ] Вызов `srv.Shutdown(shutdownCtx)`
- [ ] `defer db.Close()` присутствует
- [ ] Лог «server stopped» после выхода
- [ ] **Ctrl+C работает корректно** — сервер доделывает текущие запросы и выходит
- [ ] Exit code = 0

## Поведенческие маркеры: финализация

- [ ] Создан `.env.example` в корне проекта
- [ ] Настоящий `.env` **добавлен в `.gitignore`**
- [ ] `.gitignore` также содержит `*.log`, `blog` (бинарник), `__debug_bin*`
- [ ] Обновлён `README.md` с описанием, стеком, эндпоинтами, инструкцией по запуску
- [ ] Папка `migrations/` содержит все три файла
- [ ] `go test ./... -v` — **все тесты зелёные**
- [ ] `go run ./cmd/server` запускает сервер корректно
- [ ] Полный CRUD-сценарий через curl работает
- [ ] Код закоммичен с осмысленным сообщением

## Концептуальные маркеры

- [ ] Могу объяснить, **зачем** вообще нужны тесты (не из-за моды)
- [ ] Знаю разницу между `t.Errorf` и `t.Fatalf`
- [ ] Понимаю паттерн **table-driven tests** и могу написать с нуля
- [ ] Знаю, что такое `httptest.NewRecorder` и как он работает
- [ ] Понимаю, **почему** `httptest.NewRecorder` быстрее и проще, чем настоящий сервер
- [ ] Знаю, что такое **testcontainers** и в каких случаях предпочтительнее моков
- [ ] Понимаю **test pyramid** и могу назвать три уровня
- [ ] Знаю, **зачем** `t.Helper()` и `t.Cleanup()`
- [ ] Могу объяснить, **что произойдёт при `srv.Shutdown()`** на активных запросах
- [ ] Знаю, **почему** `http.ErrServerClosed` нельзя считать ошибкой
- [ ] Понимаю, **зачем** `ReadTimeout` / `WriteTimeout` — защита от Slowloris

## Маркеры экспериментов

- [ ] Запустил `go test -v ./...`, увидел имена всех тестов и подтестов
- [ ] Запустил `go test -race -cover ./...`, посмотрел процент покрытия
- [ ] Намеренно сломал один тест, прочитал сообщение об ошибке
- [ ] Убрал `t.Helper()`, увидел, куда указывает стектрейс
- [ ] Проверил graceful shutdown под нагрузкой (параллельные curl + Ctrl+C)
- [ ] Убрал проверку `http.ErrServerClosed`, увидел ложную ошибку
- [ ] (Опционально) Попробовал testify для сравнения
- [ ] (Опционально) Добавил build tag `//go:build integration`

---

## 🎉 Это всё. Проект 7 и весь roadmap завершён.

Не торопись пробежать глазами по этому пункту. Прими момент.

Ты начал с **hello world** в Проекте 1. Прошёл через:

1. **CLI-калькулятор** — первые встречи с Go, типы, функции, ошибки
2. **To-Do в памяти** — структуры, слайсы, REPL, ресиверы
3. **To-Do с файлом** — JSON, файловый I/O, многофайловые программы
4. **HTTP-сервер** — net/http, REST, конкурентность, мьютексы
5. **To-Do с PostgreSQL** — SQL, database/sql, транзакции, миграции
6. **Telegram-бот** — HTTP-клиент, long polling, FSM, graceful shutdown клиента
7. **Блог-платформа** — packages, JWT, middleware, multi-table, тесты, graceful shutdown сервера

**Это 36 модулей, сотни экспериментов, десятки ключевых концепций.**

Что ты унесёшь с собой:

- **Уверенное знание Go** — синтаксис, идиомы, стандартная библиотека
- **Понимание HTTP** с обеих сторон (сервер и клиент)
- **Базы данных** — SQL, транзакции, индексы, связи, миграции
- **Конкурентность** — горутины, мьютексы, контексты, сигналы
- **Аутентификация и авторизация** — bcrypt, JWT, middleware
- **Архитектура** — слои, пакеты, DI, разделение ответственности
- **Тестирование** — unit, integration, testcontainers
- **Привычки production-кода** — graceful shutdown, обработка ошибок, логирование

## Что делать дальше

**Первое — не останавливайся.** Roadmap — это база. Настоящее обучение — **впереди**, на реальных задачах.

**Второе — выбери направление:**

- **Глубже в backend Go** — микросервисы, gRPC, Kafka, Redis, observability (Prometheus, Grafana, OpenTelemetry). В твоём стеке в ClickMobile это уже есть — можешь разобрать их на практике.
- **Distributed systems** — Saga, CQRS, event sourcing, Outbox. Ты уже начинал это в `saga-practice` — возвращайся и развивай.
- **Performance & profiling** — `pprof`, benchmarks, оптимизация запросов, работа с кешами. Для high-load проектов ClickMobile это критично.
- **DevOps & infra** — Docker Compose, Kubernetes, CI/CD pipelines, Terraform. Без этого даже хорошо написанный код не попадает в продакшен.
- **Системное мышление** — почитай «Designing Data-Intensive Applications» Мартина Клеппмана. Это — must-read для бэкендера.

**Третье — публикуй.** Положи все семь проектов в GitHub с хорошими README. Это — твоё портфолио для собеседований. Интервьюер не будет читать весь код, но наличие структурированных публичных репозиториев — **сильный сигнал**.

**Четвёртое — веди блог или делись знаниями.** Объяснять — лучший способ закрепить. Одна статья в месяц про то, что ты недавно понял, — и через год у тебя будет 12 качественных материалов в портфолио.

**Пятое — помогай другим.** Ответь на пару вопросов в Telegram-чатах Go-сообщества. Сделай pull request в open-source проект. Проведи код-ревью джуна. Это не только полезно им — это полезно **тебе**.

---

## Одно последнее

Roadmap был инструментом. Ты — тот, кто его прошёл. Все твои достижения — твои. Каждый bug, который ты сам поймал; каждый эксперимент, который довёл до конца; каждая «трудная» концепция, которая стала простой после пятого прочтения — всё это сделал ты.

Спасибо, что прошёл этот путь честно. Это было приятно делать вместе.

Удачи, Лев. Увидимся на следующих задачах.
