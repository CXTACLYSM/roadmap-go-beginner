# Модуль 6. Финальная интеграция — теория

## Контекст

Это **финальный модуль** Проекта 5. Ты соединишь всё, что сделал:

- HTTP-сервер из Проекта 4
- `TaskRepository` из модулей 4-5

Результат — REST API, который хранит данные в PostgreSQL вместо файла. Это **минимальная боевая структура** современного бэкенда: HTTP + БД, разделение на слои, явное управление подключениями.

После этого модуля ты сможешь сказать «я умею писать бэкенды на Go» — и это будет правдой.

## Ядро 1: что меняется

Сравним структуру **до** и **после** интеграции.

### Было (Проект 4, in-memory + файл)

```
app/
├── main.go         ← инициализация + REPL/HTTP
├── server.go       ← обработчики
├── tasklist.go     ← TaskList с Tasks, NextID, mu
├── task.go         ← Task
└── tasks.json      ← данные
```

В `tasklist.go`:
- Поля `Tasks []Task`, `NextID int`, `mu sync.Mutex`
- Методы `Add`, `Update`, `Remove` — изменяют слайс
- Методы `Save`, `Load` — работают с файлом
- Защита через `sync.Mutex`

В `main.go`:
- Создание `TaskList`
- Загрузка из файла
- `saveOrWarn` после каждой изменяющей операции

### Стало (Проект 5, PostgreSQL)

```
005-todo-postgres/
├── go.mod
├── main.go              ← инициализация + HTTP
├── server.go            ← обработчики
├── repository.go        ← TaskRepository (бывший tasklist.go)
├── task.go              ← Task
└── migrations/
    ├── 001_create_tasks_table.sql
    ├── 002_index_done.sql
    └── 003_check_title_not_empty.sql
```

В `repository.go`:
- Поле `db *sql.DB`
- Методы `GetAll`, `GetByID`, `Add`, `Update`, `Remove` — работают с БД через SQL
- **Никакого `Save`, `Load`** — БД сама знает, как сохранять
- **Никакого `sync.Mutex`** — БД сама управляет конкурентностью
- **Никакого `NextID`** — `BIGSERIAL` сам генерирует id

В `main.go`:
- Подключение к БД через `sql.Open` + `Ping`
- Передача `*sql.DB` в репозиторий через конструктор
- **Никакого `saveOrWarn`** — каждый запрос уже коммитится в БД

Это **сильное упрощение**. Меньше состояния, меньше ручной работы, меньше мест для ошибок.

## Ядро 2: новый main

```go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "os"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        log.Fatal("DATABASE_URL не задан")
    }

    db, err := sql.Open("pgx", dsn)
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

    log.Println("Подключение к PostgreSQL установлено")

    repo := NewTaskRepository(db)
    server := &Server{tasks: repo}

    http.HandleFunc("GET /tasks", server.listTasks)
    http.HandleFunc("GET /tasks/{id}", server.getTask)
    http.HandleFunc("POST /tasks", server.createTask)
    http.HandleFunc("PUT /tasks/{id}", server.updateTask)
    http.HandleFunc("DELETE /tasks/{id}", server.deleteTask)

    log.Println("Сервер запущен на http://localhost:8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatalf("http.ListenAndServe: %v", err)
    }
}
```

Шаги, которые делает `main`:

1. Читает `DATABASE_URL`
2. Открывает пул соединений через `sql.Open`
3. Настраивает пул (Open/Idle/Lifetime)
4. Пингует, чтобы убедиться, что БД доступна
5. Создаёт репозиторий, передавая `db`
6. Создаёт `Server`, передавая репозиторий
7. Регистрирует обработчики
8. Запускает HTTP-сервер

**`db.SetMaxOpenConns(25)`** — это важный момент. Без ограничения сервер под нагрузкой может открыть тысячи соединений и положить PostgreSQL. 25 — разумное значение для среднего сервера.

## Ядро 3: новый Server

`Server` теперь хранит репозиторий, а не `*TaskList`:

```go
type Server struct {
    tasks *TaskRepository
}
```

И каждый обработчик использует методы репозитория:

```go
func (s *Server) listTasks(w http.ResponseWriter, r *http.Request) {
    tasks, err := s.tasks.GetAll()
    if err != nil {
        log.Printf("GetAll error: %v", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    if err := json.NewEncoder(w).Encode(tasks); err != nil {
        log.Printf("encode error: %v", err)
    }
}
```

Заметь две новых вещи:

1. **`GetAll` теперь возвращает ошибку** — потому что обращение к БД может упасть. Если падает — возвращаем 500 (это **наша** проблема, не клиента).
2. **Логируем ошибку**, прежде чем возвращать 500. Клиенту мы не показываем детали (это безопасность), но себе в логах оставляем.

Полный список обработчиков идентичен Проекту 4, только все вызовы методов идут через `s.tasks` (репозиторий) и **обрабатывают ошибки**.

## Ядро 4: разделение «технических» и «бизнес» ошибок

В предыдущих проектах ошибки были простые: «не найдено» или «плохие данные». Теперь добавляются **технические** ошибки от БД: разрыв соединения, deadlock, нехватка памяти, и так далее.

Правило:

| Тип ошибки | HTTP-статус | Что в логе | Что клиенту |
|------------|-------------|------------|-------------|
| Невалидный ввод (пустой title, плохой JSON) | 400 | — | детали ошибки |
| Не найдено | 404 | — | "task not found" |
| Техническая ошибка БД | 500 | полная ошибка | "internal error" |

**Никогда не показывай клиенту полные сообщения от БД.** Это утечка информации (имена таблиц, версия PostgreSQL, иногда даже фрагменты данных).

Пример:

```go
task, err := s.tasks.GetByID(id)
if err != nil {
    if errors.Is(err, sql.ErrNoRows) {
        http.Error(w, "task not found", http.StatusNotFound)
        return
    }
    log.Printf("GetByID error: %v", err)
    http.Error(w, "internal error", http.StatusInternalServerError)
    return
}
```

⚠️ **Тонкость**: твой `GetByID` возвращает обёрнутую ошибку (`fmt.Errorf("...: %w", sql.ErrNoRows)`). Чтобы `errors.Is` сработал, **обёртывание через `%w` обязательно**. Если ты использовал `%s` или просто `fmt.Errorf("not found")` — `errors.Is` не сработает.

## Ядро 5: что убирается

В Проекте 4 у тебя были:

- **`sync.Mutex`** в `TaskList` → **убираем**, БД сама обрабатывает параллелизм
- **`Save`/`Load`** методы → **убираем**, данные всегда в БД
- **Helper `saveOrWarn`** → **убираем**, никакой явной записи не нужно
- **Глобальная константа `dataFile`** → **убираем**, её роль играет `DATABASE_URL`
- **Загрузка данных в `main`** → **убираем**, БД и так живая
- **`copy` в `All`** → **убираем**, нет общей памяти, копировать нечего

Это очень освобождает. Меньше кода — меньше багов. И всё это благодаря тому, что **ответственность за данные теперь у БД**, а не у твоего Go-кода.

## Ядро 6: что добавляется

- **Обработка ошибок БД во всех обработчиках** — потому что любой запрос к БД может упасть
- **Логирование технических ошибок** — без него ты не узнаешь, что в продакшене что-то не так
- **Пинг при старте** — чтобы упасть на старте, если БД недоступна
- **Настройка пула соединений** — для устойчивости под нагрузкой
- **`migrations/` папка** — часть проекта

## Ядро 7: запуск

Полная последовательность:

```bash
# 1. Запустить контейнер с БД
docker run -d --name todo-pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -v todo-pg-data:/var/lib/postgresql/data postgres:16
sleep 3

# 2. Применить миграции
export PGPASSWORD=secret
for file in migrations/*.sql; do
    psql -h localhost -U postgres -d postgres -f "$file"
done

# 3. Запустить сервер
DATABASE_URL="postgres://postgres:secret@localhost:5432/postgres?sslmode=disable" go run .
```

В другом терминале:

```bash
# Тестируем
curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" -d '{"title":"Купить молоко"}'
curl http://localhost:8080/tasks
```

Останавливаешь сервер (Ctrl+C), запускаешь снова — данные на месте, потому что они в БД, а не в памяти.

**Запускаешь два экземпляра** на разных портах:

```bash
# Терминал 1
PORT=8080 DATABASE_URL=... go run .

# Терминал 2 (поменяй порт в коде или сделай его конфигурируемым)
PORT=8081 DATABASE_URL=... go run .
```

Оба видят **одни и те же данные**, и работают корректно благодаря тому, что БД управляет конкурентностью. Это то, что в Проекте 4 ломалось.

## Ядро 8: контексты

В реальном продакшен-коде каждый запрос к БД идёт **с контекстом**:

```go
func (r *TaskRepository) GetAllContext(ctx context.Context) ([]Task, error) {
    rows, err := r.db.QueryContext(ctx, "SELECT id, title, done FROM tasks ORDER BY id")
    // ...
}
```

И в обработчике:

```go
tasks, err := s.tasks.GetAllContext(r.Context())
```

`r.Context()` — это контекст HTTP-запроса. Он отменяется, когда:
- Клиент закрыл соединение
- Истёк дедлайн (если задан middleware-ом)
- `http.Server` вызвал shutdown

С контекстом БД-запрос **отменится автоматически**, если клиент уже не ждёт ответа. Это экономит ресурсы.

**Для модуля 6 — добавь контексты ко всем методам репозитория.** Это будет последним штрихом, который делает код «продакшен-готовым».

## Ядро 9: финальная структура

```
005-todo-postgres/
├── go.mod
├── go.sum
├── main.go              ← инициализация
├── server.go            ← Server и обработчики
├── repository.go        ← TaskRepository
├── task.go              ← Task
└── migrations/
    ├── 001_create_tasks_table.sql
    ├── 002_index_done.sql
    └── 003_check_title_not_empty.sql
```

Это — структура, очень близкая к тому, что ты увидишь в реальных Go-проектах. С двумя уточнениями (которые придут в Проектах 6-7):

- **Пакеты** вместо `package main` для всего: `internal/storage`, `internal/api`, `cmd/server/main.go`
- **Бизнес-логика** в отдельном слое (service), между handler и repository

Сейчас всё в одном пакете — это нормально для маленького проекта. Добавление пакетов — следующий шаг.

## Вопросы на понимание

1. Что **убирается** из кода после перехода с файла на БД?
2. Почему **больше не нужен** `sync.Mutex` в репозитории?
3. Что **добавляется** в обработчики из-за БД?
4. Зачем устанавливать `db.SetMaxOpenConns`?
5. **Почему нельзя** показывать клиенту полные сообщения от БД?
6. Чем `r.Context()` помогает при работе с БД?
7. Что произойдёт, если запустить **два экземпляра** сервера на одну БД? Будут ли проблемы?
