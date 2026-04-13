# Модуль 4. PostgreSQL для бота — теория

## Контекст

В модуле 3 бот функционально работает, но данные в памяти — пропадают при перезапуске. Сейчас ты заменишь in-memory `Storage` на `Repository` поверх PostgreSQL.

Главное отличие от Проекта 5: теперь у тебя **много пользователей**, и каждое сообщение должно работать **только с данными своего пользователя**. Это требует фильтрации по `chat_id` в каждом SQL-запросе.

Этот модуль — практика **переиспользования** того, что ты уже умеешь:

- `database/sql + pgx` — из Проекта 5
- Миграции — из Проекта 5
- Подключение через `DATABASE_URL` — из Проекта 5
- Структура `Repository` с методами CRUD — из Проекта 5

Новое здесь — **multi-tenant**: одна таблица для всех пользователей, фильтрация по `chat_id`.

## Ядро 1: схема таблицы для multi-user

```sql
CREATE TABLE tasks (
    id          BIGSERIAL PRIMARY KEY,
    chat_id     BIGINT NOT NULL,
    user_task_id BIGINT NOT NULL,
    title       TEXT NOT NULL,
    done        BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_chat_id ON tasks (chat_id);
CREATE UNIQUE INDEX idx_tasks_chat_user_id ON tasks (chat_id, user_task_id);
```

Несколько важных моментов:

1. **`id BIGSERIAL`** — глобальный технический id (как в Проекте 5). Уникален по всей таблице, неинтересен пользователю.

2. **`chat_id BIGINT`** — id чата (Telegram user). Не PRIMARY KEY (может повторяться — у одного пользователя много задач), но будет в индексе для быстрого поиска.

3. **`user_task_id BIGINT`** — это **id, который видит пользователь**. Для Лева это 1, 2, 3. Для Ани — тоже 1, 2, 3. Они не конфликтуют, потому что есть фильтр по `chat_id`.

4. **`UNIQUE (chat_id, user_task_id)`** — гарантирует, что у одного пользователя не будет двух задач с одним id. Это — **составной уникальный индекс** (composite unique).

5. **`INDEX (chat_id)`** — для быстрого поиска всех задач одного пользователя (`SELECT ... WHERE chat_id = $1`).

### Альтернативный дизайн (упрощённый)

Можно было бы использовать один глобальный `id` (без `user_task_id`), и показывать пользователю **позицию в списке** вместо id. Но тогда позиции «плавают» при удалении (удалил вторую — третья стала второй), и команды `/done 3` и `/delete 3` становятся неоднозначными.

Дизайн с per-user `user_task_id` — стандартный для многопользовательских приложений. Он чуть сложнее в реализации (нужно вычислять следующий id), но сильно удобнее для пользователя.

## Ядро 2: вычисление user_task_id

Главная сложность: при создании новой задачи у пользователя `chat_id=12345` нужно понять, какой `user_task_id` ему присвоить. Варианты:

### Вариант A: SELECT MAX + 1

```sql
INSERT INTO tasks (chat_id, user_task_id, title)
VALUES (
    $1,
    COALESCE((SELECT MAX(user_task_id) FROM tasks WHERE chat_id = $1), 0) + 1,
    $2
)
RETURNING id, chat_id, user_task_id, title, done;
```

Один SQL-запрос. Внутри подзапрос находит максимальный `user_task_id` для этого пользователя (или 0, если задач нет — это работа `COALESCE`), прибавляет 1, и вставляет.

⚠️ **Проблема race condition:** если два запроса одновременно вычислят `MAX = 5` и оба попытаются вставить с `user_task_id = 6`, второй упадёт из-за UNIQUE constraint. Это **редкий, но возможный сценарий**.

### Вариант B: транзакция с SELECT FOR UPDATE

```sql
BEGIN;
SELECT MAX(user_task_id) FROM tasks WHERE chat_id = $1 FOR UPDATE;
INSERT INTO tasks (chat_id, user_task_id, title) VALUES ($1, ?, $2) RETURNING ...;
COMMIT;
```

`FOR UPDATE` блокирует строки, чтобы другой транзакции пришлось ждать. Решает race condition, но добавляет накладные расходы.

### Вариант C: отдельная таблица счётчиков

```sql
CREATE TABLE chat_counters (
    chat_id BIGINT PRIMARY KEY,
    next_task_id BIGINT NOT NULL DEFAULT 1
);

-- При INSERT задачи:
UPDATE chat_counters SET next_task_id = next_task_id + 1 WHERE chat_id = $1 RETURNING next_task_id;
-- ... потом INSERT с этим next_task_id
```

Это правильный подход для production, но требует двух запросов и дополнительной таблицы.

### Что выбираем?

**Для нашего проекта — Вариант A (`COALESCE + MAX + 1`).** Race condition теоретически возможна, но в чате одного пользователя сообщения приходят **строго последовательно** (Telegram гарантирует порядок), поэтому два одновременных INSERT для одного `chat_id` в нашем сценарии **невозможны**.

Если бы мы делали публичный API, которым можно вызвать `Add` извне Telegram — нужен был бы Вариант B или C.

## Ядро 3: миграция

```sql
-- migrations/001_create_tasks_table.sql
CREATE TABLE tasks (
    id           BIGSERIAL PRIMARY KEY,
    chat_id      BIGINT NOT NULL,
    user_task_id BIGINT NOT NULL,
    title        TEXT NOT NULL,
    done         BOOLEAN NOT NULL DEFAULT false,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_chat_id ON tasks (chat_id);
CREATE UNIQUE INDEX idx_tasks_chat_user_task_id ON tasks (chat_id, user_task_id);
```

Применение — как в Проекте 5:

```bash
psql -h localhost -U postgres -d postgres -f migrations/001_create_tasks_table.sql
```

⚠️ **Используй отдельную БД для бота**, не ту, которая была для Проекта 5. Иначе таблица `tasks` пересечётся со старой. Вариант:

```bash
psql -h localhost -U postgres -d postgres -c "CREATE DATABASE todo_bot;"
psql -h localhost -U postgres -d todo_bot -f migrations/001_create_tasks_table.sql
```

И в `DATABASE_URL` указать `/todo_bot`.

## Ядро 4: тип Repository

Аналогично Проекту 5, но с `chat_id` в каждом запросе:

```go
type Repository struct {
    db *sql.DB
}

func NewRepository(db *sql.DB) *Repository {
    return &Repository{db: db}
}

func (r *Repository) Add(chatID int64, title string) (Task, error) {
    var t Task
    err := r.db.QueryRow(`
        INSERT INTO tasks (chat_id, user_task_id, title)
        VALUES (
            $1,
            COALESCE((SELECT MAX(user_task_id) FROM tasks WHERE chat_id = $1), 0) + 1,
            $2
        )
        RETURNING id, chat_id, user_task_id, title, done
    `, chatID, title).Scan(&t.ID, &t.ChatID, &t.UserTaskID, &t.Title, &t.Done)
    if err != nil {
        return Task{}, fmt.Errorf("insert: %w", err)
    }
    return t, nil
}

func (r *Repository) List(chatID int64) ([]Task, error) {
    rows, err := r.db.Query(`
        SELECT id, chat_id, user_task_id, title, done
        FROM tasks
        WHERE chat_id = $1
        ORDER BY user_task_id
    `, chatID)
    if err != nil {
        return nil, fmt.Errorf("query: %w", err)
    }
    defer rows.Close()

    tasks := []Task{}
    for rows.Next() {
        var t Task
        if err := rows.Scan(&t.ID, &t.ChatID, &t.UserTaskID, &t.Title, &t.Done); err != nil {
            return nil, fmt.Errorf("scan: %w", err)
        }
        tasks = append(tasks, t)
    }
    return tasks, rows.Err()
}

func (r *Repository) Done(chatID int64, userTaskID int64) error {
    result, err := r.db.Exec(`
        UPDATE tasks SET done = true
        WHERE chat_id = $1 AND user_task_id = $2
    `, chatID, userTaskID)
    if err != nil {
        return fmt.Errorf("update: %w", err)
    }

    affected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("rows affected: %w", err)
    }
    if affected == 0 {
        return fmt.Errorf("задача не найдена: %d", userTaskID)
    }
    return nil
}

func (r *Repository) Delete(chatID int64, userTaskID int64) error {
    result, err := r.db.Exec(`
        DELETE FROM tasks
        WHERE chat_id = $1 AND user_task_id = $2
    `, chatID, userTaskID)
    if err != nil {
        return fmt.Errorf("delete: %w", err)
    }

    affected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("rows affected: %w", err)
    }
    if affected == 0 {
        return fmt.Errorf("задача не найдена: %d", userTaskID)
    }
    return nil
}
```

Заметь:

1. **`Task` теперь имеет поле `UserTaskID`** — то, что видит пользователь
2. **Каждый запрос содержит `WHERE chat_id = $1`** — это **неотъемлемая** часть multi-tenant логики
3. **`ORDER BY user_task_id`** в `List` — чтобы порядок был стабильным
4. **`RowsAffected == 0`** — то же, что в Проекте 5: означает «не нашли»

## Ядро 5: критическая важность фильтра по chat_id

**Это самое важное правило мульти-тенант систем.** Каждый запрос **обязан** содержать фильтр по «владельцу» данных. Если ты забудешь:

```go
// КАТАСТРОФА
func (r *Repository) Done(chatID int64, userTaskID int64) error {
    result, err := r.db.Exec(`
        UPDATE tasks SET done = true
        WHERE user_task_id = $1
    `, userTaskID)
    // ...
}
```

— Лев напишет `/done 1`, и **у каждого пользователя** с задачей `user_task_id = 1` она станет выполненной. Это **утечка** между пользователями, эквивалент SQL-injection по последствиям.

В реальных системах это называется **IDOR** (Insecure Direct Object Reference) — одна из топ-уязвимостей OWASP. И решение всегда одно: **в каждом запросе фильтруй по владельцу**.

В нашем проекте «владелец» — `chat_id`. В Проекте 7 (блог-платформа) это будет `user_id`, и фильтр будет `WHERE user_id = $1`.

**Привычка дисциплины:** перед каждым `r.db.Query/Exec` мысленно спроси себя: «есть ли в WHERE ключ владельца?». Если нет — возможно, это бэкдор.

## Ядро 6: тип Task

```go
type Task struct {
    ID         int64
    ChatID     int64
    UserTaskID int64
    Title      string
    Done       bool
}
```

Заметь два разных id:

- **`ID`** — глобальный (BIGSERIAL). Используется во внутренней логике, не показывается пользователю.
- **`UserTaskID`** — то, что видит пользователь. Это и есть тот «номер задачи», который пользователь набирает в `/done 3`.

## Ядро 7: интеграция в Bot

Тип `Bot` теперь хранит `*Repository` вместо `*Storage`:

```go
type Bot struct {
    tg   *TelegramClient
    repo *Repository
}

func NewBot(tg *TelegramClient, repo *Repository) *Bot {
    return &Bot{tg: tg, repo: repo}
}
```

Handle-функции изменяются минимально — заменяется тип хранилища и обрабатываются ошибки от БД:

```go
func (b *Bot) handleAdd(chatID int64, arg string) {
    arg = strings.TrimSpace(arg)
    if arg == "" {
        b.send(chatID, "Использование: /add <текст задачи>")
        return
    }
    task, err := b.repo.Add(chatID, arg)
    if err != nil {
        log.Printf("Add error: %v", err)
        b.send(chatID, "Ошибка при добавлении задачи. Попробуй позже.")
        return
    }
    b.send(chatID, fmt.Sprintf("Задача #%d добавлена: %s", task.UserTaskID, task.Title))
}

func (b *Bot) handleList(chatID int64) {
    tasks, err := b.repo.List(chatID)
    if err != nil {
        log.Printf("List error: %v", err)
        b.send(chatID, "Ошибка при получении списка. Попробуй позже.")
        return
    }
    if len(tasks) == 0 {
        b.send(chatID, "Список пуст. Добавь задачу через /add")
        return
    }

    var sb strings.Builder
    for _, t := range tasks {
        check := " "
        if t.Done {
            check = "x"
        }
        fmt.Fprintf(&sb, "%d. [%s] %s\n", t.UserTaskID, check, t.Title)
    }
    b.send(chatID, sb.String())
}

func (b *Bot) handleDone(chatID int64, arg string) {
    id, err := parseTaskID(arg)
    if err != nil {
        b.send(chatID, "Использование: /done <id>")
        return
    }
    if err := b.repo.Done(chatID, id); err != nil {
        b.send(chatID, err.Error())
        return
    }
    b.send(chatID, fmt.Sprintf("Задача #%d отмечена выполненной", id))
}
```

Заметь: **сообщение для пользователя содержит `UserTaskID`**, не `ID`. Это важно — пользователь должен видеть «свои» номера, а не глобальные.

## Ядро 8: main с подключением к БД

```go
func main() {
    token := os.Getenv("BOT_TOKEN")
    if token == "" {
        log.Fatal("BOT_TOKEN не задан")
    }

    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        log.Fatal("DATABASE_URL не задан")
    }

    db, err := sql.Open("pgx", dsn)
    if err != nil {
        log.Fatalf("sql.Open: %v", err)
    }
    defer db.Close()

    db.SetMaxOpenConns(10)
    db.SetMaxIdleConns(2)
    db.SetConnMaxLifetime(5 * time.Minute)

    if err := db.Ping(); err != nil {
        log.Fatalf("db.Ping: %v", err)
    }

    log.Println("Подключение к PostgreSQL установлено")

    tg := NewTelegramClient(token)
    repo := NewRepository(db)
    bot := NewBot(tg, repo)
    bot.Run()
}
```

Это то же самое, что в Проекте 5, плюс инициализация бота. Никаких новых концепций — просто комбинация.

## Ядро 9: что убирается из кода модуля 3

- **Тип `Storage`** → заменён на `Repository`
- **`tasksByChat map[int64][]Task`** → удалено
- **`nextIDByChat`** → удалено (теперь в БД)
- **`sync.Mutex` в Storage** → не нужен, БД сама управляет конкурентностью
- **Возврат копии** → не нужен, `tasks` уже копия из `Scan`

И добавляется:

- **Подключение к БД** в `main`
- **Папка `migrations/`**
- **Импорт `database/sql` и драйвера**
- **Обработка ошибок БД** во всех handle-функциях

## Вопросы на понимание

1. Почему таблица содержит **два** поля id (`id` и `user_task_id`)? Зачем второй?
2. Что произойдёт, если забыть `WHERE chat_id = $1` в запросе?
3. Что такое **IDOR** и как фильтрация по `chat_id` от него защищает?
4. Зачем нужен составной `UNIQUE INDEX (chat_id, user_task_id)`?
5. Как вычисляется `user_task_id` для новой задачи? Какие альтернативы?
6. Почему мы используем именно Вариант A (COALESCE + MAX + 1), а не транзакцию с FOR UPDATE?
7. Что **убирается** из кода после перехода с in-memory storage на БД?
8. Какое поле задачи показывается пользователю — `ID` или `UserTaskID`?
