# Модуль 6. Финальная интеграция — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Что должно быть в `main.go` **до** запуска HTTP-сервера?
2. Какое поле должно быть у `Server` вместо `*TaskList`?
3. Какие три типа ошибок может вернуть метод репозитория? Какой статус для каждого?
4. Зачем нужен `db.SetMaxOpenConns`?
5. Что произойдёт, если убрать `defer db.Close()`?

## Уровень 2: про порядок инициализации в main

Безопасный порядок:

```go
func main() {
    // 1. Конфигурация
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        log.Fatal("DATABASE_URL не задан")
    }

    // 2. Подключение к БД
    db, err := sql.Open("pgx", dsn)
    if err != nil {
        log.Fatalf("sql.Open: %v", err)
    }
    defer db.Close()

    // 3. Настройка пула
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)

    // 4. Проверка
    if err := db.Ping(); err != nil {
        log.Fatalf("db.Ping: %v", err)
    }

    // 5. Создание зависимостей
    repo := NewTaskRepository(db)
    server := &Server{tasks: repo}

    // 6. Регистрация эндпоинтов
    http.HandleFunc("GET /tasks", server.listTasks)
    http.HandleFunc("GET /tasks/{id}", server.getTask)
    http.HandleFunc("POST /tasks", server.createTask)
    http.HandleFunc("PUT /tasks/{id}", server.updateTask)
    http.HandleFunc("DELETE /tasks/{id}", server.deleteTask)

    // 7. Запуск
    log.Println("Сервер запущен на http://localhost:8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatalf("http.ListenAndServe: %v", err)
    }
}
```

Если нарушить порядок (например, регистрация до создания репозитория) — компилятор поймает не всегда. Лучше держать структуру строгой.

## Уровень 3: про обработчик с обработкой ошибок БД

Шаблон, который повторяется во всех обработчиках:

```go
func (s *Server) getTask(w http.ResponseWriter, r *http.Request) {
    id, ok := parseID(w, r)
    if !ok {
        return
    }

    task, err := s.tasks.GetByID(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            http.Error(w, "task not found", http.StatusNotFound)
            return
        }
        log.Printf("GetByID(%d) error: %v", id, err)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    writeJSON(w, http.StatusOK, task)
}
```

Структура такая:

1. Распарси входные данные (`parseID`, `Decode` для тела) → 400 при ошибке
2. Вызови репозиторий
3. Обработай ошибку:
   - **`sql.ErrNoRows`** или твоя ошибка «не найдено» → 404
   - **Любая другая** → лог + 500
4. На успехе → 200/201/204 + данные

⚠️ **Тонкость**: чтобы `errors.Is(err, sql.ErrNoRows)` сработал, твой репозиторий должен **обернуть** ошибку через `%w`:

```go
// В репозитории:
if errors.Is(err, sql.ErrNoRows) {
    return Task{}, fmt.Errorf("not found: %w", err)  // ← %w!
}
```

Если ты возвращаешь `fmt.Errorf("not found")` без `%w` — обёртка теряет связь с оригиналом, и `errors.Is` не сработает.

**Альтернатива** — определить свою sentinel ошибку:

```go
// В пакете repository
var ErrNotFound = errors.New("task not found")

// В методе
if errors.Is(err, sql.ErrNoRows) {
    return Task{}, ErrNotFound
}

// В обработчике
if errors.Is(err, ErrNotFound) {
    http.Error(w, "task not found", http.StatusNotFound)
    return
}
```

Это **более чистый подход** для большего проекта. Для модуля 6 можешь использовать любой.

## Уровень 4: про writeJSON helper

Чтобы не дублировать `Header().Set("Content-Type", "application/json") + WriteHeader + Encode` в каждом обработчике:

```go
func writeJSON(w http.ResponseWriter, status int, value any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(value); err != nil {
        log.Printf("encode error: %v", err)
    }
}
```

Использование:

```go
writeJSON(w, http.StatusOK, tasks)
writeJSON(w, http.StatusCreated, newTask)
```

Для DELETE с 204 — без тела:

```go
w.WriteHeader(http.StatusNoContent)
```

## Уровень 5: про парсинг ID из path

Helper из Проекта 4:

```go
func parseID(w http.ResponseWriter, r *http.Request) (int64, bool) {
    idStr := r.PathValue("id")
    id, err := strconv.ParseInt(idStr, 10, 64)
    if err != nil {
        http.Error(w, "invalid id", http.StatusBadRequest)
        return 0, false
    }
    return id, true
}
```

Заметь: возвращаем **`int64`** (для `BIGSERIAL`), используем `ParseInt(... 10, 64)` вместо `Atoi` — потому что `Atoi` возвращает `int`, и его потом надо конвертировать.

## Уровень 6: про DTO и валидацию

```go
func (s *Server) createTask(w http.ResponseWriter, r *http.Request) {
    var input struct {
        Title string `json:"title"`
    }

    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        http.Error(w, "invalid json", http.StatusBadRequest)
        return
    }

    if strings.TrimSpace(input.Title) == "" {
        http.Error(w, "title is required", http.StatusBadRequest)
        return
    }

    task, err := s.tasks.Add(input.Title)
    if err != nil {
        log.Printf("Add error: %v", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    writeJSON(w, http.StatusCreated, task)
}
```

Идея та же, что в Проекте 4: **валидируй ввод в Go**, **перед** обращением к БД. Это даёт понятные сообщения клиенту и не нагружает БД заведомо плохими запросами.

## Уровень 7: про настройку пула

```go
db.SetMaxOpenConns(25)         // максимум активных соединений
db.SetMaxIdleConns(5)          // максимум idle (готовых, но неиспользуемых)
db.SetConnMaxLifetime(5 * time.Minute)
```

Цифры — это не догма, а разумные значения для среднего сервера. В реальной нагрузке ты бы их тюнил под свой кейс.

**Без этих настроек** `database/sql` использует `MaxOpenConns = 0` (бесконечно) — и под нагрузкой может открыть тысячи соединений, исчерпать лимит PostgreSQL (`max_connections = 100` по умолчанию) и положить БД.

**Поэтому всегда** ставь `SetMaxOpenConns` в продакшен-сервере, даже если не уверен, какое значение правильное.

## Уровень 8: про PORT через переменную окружения (опционально)

```go
port := os.Getenv("PORT")
if port == "" {
    port = "8080"
}

log.Printf("Сервер запущен на http://localhost:%s", port)
if err := http.ListenAndServe(":"+port, nil); err != nil {
    log.Fatalf("http.ListenAndServe: %v", err)
}
```

Это позволяет запускать **несколько экземпляров** на разных портах:

```bash
PORT=8080 go run . &
PORT=8081 go run . &
```

И проверить, что они оба работают с одной БД.

## Если совсем тупик

Скажи Claude конкретно: «модуль 6 проекта 5, [конкретная проблема]». Самые частые ситуации:

- **«errors.Is(err, sql.ErrNoRows) не срабатывает»** → в репозитории ошибка обёрнута через `%s` или `%v` вместо `%w`
- **«500 на каждый запрос»** → проверь логи сервера, обычно там полная ошибка от БД
- **«too many connections»** → не настроил `SetMaxOpenConns`, или не закрываешь `rows`
- **«connection refused при первом запросе»** → БД не запущена или порт неправильный
- **«invalid id» на нормальном id** → используешь `ParseInt(... 10, 64)`, проверь radix=10 и bitsize=64
