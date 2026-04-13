# Модуль 4. PostgreSQL для бота — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какое поле должно быть в **каждом запросе** WHERE для multi-tenant?
2. Зачем нужно поле `user_task_id` отдельно от `id`?
3. Какой паттерн SQL вычисляет следующий `user_task_id` в одном запросе?
4. Как обнаружить «не найдено» в `Done`/`Delete`?
5. Почему **больше не нужен** мьютекс?

## Уровень 2: про DDL таблицы

```sql
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

Несколько моментов:

- **`chat_id BIGINT`** — Telegram chat id. Не PRIMARY KEY (повторяется), но в индексе.
- **Составной UNIQUE INDEX `(chat_id, user_task_id)`** — гарантирует уникальность пары. У одного `chat_id` не будет двух задач с одним `user_task_id`.
- **`INDEX (chat_id)`** — отдельно для быстрого поиска по `chat_id`. (На самом деле составной UNIQUE индекс уже покрывает поиск по первой колонке `chat_id`, так что отдельный индекс необязателен. Но для ясности можно оставить.)

## Уровень 3: про SQL для Add

Самый интересный запрос модуля:

```sql
INSERT INTO tasks (chat_id, user_task_id, title)
VALUES (
    $1,
    COALESCE((SELECT MAX(user_task_id) FROM tasks WHERE chat_id = $1), 0) + 1,
    $2
)
RETURNING id, chat_id, user_task_id, title, done;
```

Разбираем:

- **`$1`** в `chat_id` — первый параметр (chatID)
- **`COALESCE(..., 0)`** — если подзапрос вернул NULL (нет задач), вернуть 0
- **`(SELECT MAX(user_task_id) FROM tasks WHERE chat_id = $1)`** — найти максимальный `user_task_id` для **этого** пользователя
- **`+ 1`** — следующий после максимального
- **`$2`** в `title` — второй параметр (title)
- **`RETURNING ...`** — возврат вставленной строки

Заметь: `$1` используется **дважды**. PostgreSQL это умеет — один параметр ссылается из нескольких мест.

## Уровень 4: про Scan с пятью полями

```go
err := r.db.QueryRow(`...`, chatID, title).Scan(
    &t.ID,
    &t.ChatID,
    &t.UserTaskID,
    &t.Title,
    &t.Done,
)
```

Порядок аргументов `Scan` **должен совпадать** с порядком колонок в `RETURNING`. Если перепутаешь — получишь странные данные или ошибку типа.

## Уровень 5: про List с фильтром

```go
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

    tasks := []Task{}  // ← пустой слайс, не nil
    for rows.Next() {
        var t Task
        if err := rows.Scan(&t.ID, &t.ChatID, &t.UserTaskID, &t.Title, &t.Done); err != nil {
            return nil, fmt.Errorf("scan: %w", err)
        }
        tasks = append(tasks, t)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("rows iter: %w", err)
    }

    return tasks, nil
}
```

Обязательные элементы:
- **`WHERE chat_id = $1`** — фильтр
- **`ORDER BY user_task_id`** — стабильный порядок
- **`defer rows.Close()`** — не утекать соединениями
- **`rows.Err()`** после цикла
- **Явный `[]Task{}`** вместо `nil` — для удобства вызывающего кода

## Уровень 6: про Done и Delete с двумя условиями

Оба метода — UPDATE/DELETE с двумя условиями в WHERE:

```go
func (r *Repository) Done(chatID int64, userTaskID int64) error {
    result, err := r.db.Exec(
        "UPDATE tasks SET done = true WHERE chat_id = $1 AND user_task_id = $2",
        chatID, userTaskID,
    )
    // ...
}
```

**`AND`** — обе условия должны быть истинны. Это гарантирует, что мы обновляем задачу **именно у этого пользователя**, а не у любого другого с таким же `user_task_id`.

И не забудь проверку `RowsAffected`:

```go
affected, err := result.RowsAffected()
if err != nil {
    return fmt.Errorf("rows affected: %w", err)
}
if affected == 0 {
    return fmt.Errorf("задача не найдена: %d", userTaskID)
}
return nil
```

## Уровень 7: про обновление handle-функций

Главное изменение — обработка ошибок от БД. Шаблон:

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
        b.send(chatID, "Ошибка при добавлении. Попробуй позже.")
        return
    }
    b.send(chatID, fmt.Sprintf("Задача #%d добавлена: %s", task.UserTaskID, task.Title))
}
```

Три исхода:
- **Невалидный ввод (пустой arg)** → подсказка
- **Ошибка БД** → лог + общее сообщение
- **Успех** → подтверждение с `task.UserTaskID`

`task.UserTaskID`, **не** `task.ID`. Пользователь должен видеть свои номера, а не глобальные.

## Уровень 8: про обновление main

Структура `main`:

```go
func main() {
    // 1. Конфиг
    token := os.Getenv("BOT_TOKEN")
    if token == "" { log.Fatal("BOT_TOKEN не задан") }
    
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" { log.Fatal("DATABASE_URL не задан") }

    // 2. БД
    db, err := sql.Open("pgx", dsn)
    if err != nil { log.Fatalf("sql.Open: %v", err) }
    defer db.Close()

    db.SetMaxOpenConns(10)
    db.SetMaxIdleConns(2)
    db.SetConnMaxLifetime(5 * time.Minute)

    if err := db.Ping(); err != nil { log.Fatalf("db.Ping: %v", err) }
    log.Println("Подключение к PostgreSQL установлено")

    // 3. Зависимости
    tg := NewTelegramClient(token)
    repo := NewRepository(db)
    bot := NewBot(tg, repo)

    // 4. Запуск
    bot.Run()
}
```

Помни порядок: конфиг → БД → пинг → зависимости → запуск.

## Уровень 9: про создание отдельной БД

Чтобы не пересекаться с Проектом 5, создай отдельную БД:

```bash
psql -h localhost -U postgres -d postgres -c "CREATE DATABASE todo_bot;"
```

И в `DATABASE_URL` укажи `/todo_bot` вместо `/postgres`:

```
postgres://postgres:secret@localhost:5432/todo_bot?sslmode=disable
```

Иначе таблица `tasks` будет иметь старую схему из Проекта 5 (без `chat_id`), и запросы будут падать.

## Если совсем тупик

Скажи Claude конкретно: «модуль 4 проекта 6, [конкретная проблема]». Самые частые ситуации:

- **«У одного пользователя задачи всех других»** → забыл `WHERE chat_id = $1` в `List`
- **«duplicate key value violates unique constraint»** → race condition в `Add` (очень редко) или забыл подзапрос с `MAX`
- **«column "user_task_id" does not exist»** → забыл применить миграцию или применил к старой БД
- **«panic: runtime error: invalid memory address»** → пытаешься обратиться к `task.UserTaskID`, но `Scan` его не заполнил (опечатка в SELECT)
- **«too many connections»** → не настроил `SetMaxOpenConns`
