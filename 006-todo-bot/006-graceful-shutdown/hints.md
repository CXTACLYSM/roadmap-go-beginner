# Модуль 6. Graceful shutdown — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какой пакет Go отвечает за сигналы?
2. Какая функция регистрирует канал для сигналов?
3. Что такое `context.Context` в одной фразе?
4. Какая функция создаёт контекст с возможностью отмены?
5. Почему контекст должен быть **первым** параметром?

## Уровень 2: про signal.Notify

```go
import (
    "os"
    "os/signal"
    "syscall"
)

sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
```

Несколько моментов:

- **Буфер 1** в канале — обязателен. Без него, если сигнал придёт до `<-sigChan`, он пропадёт (Go не блокирует на отправке в `signal.Notify`)
- **`os.Interrupt`** — кросс-платформенный вариант SIGINT (Ctrl+C)
- **`syscall.SIGTERM`** — нужен для kubernetes/systemd/docker stop

## Уровень 3: про context.WithCancel

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

- **`context.Background()`** — пустой root-контекст, начало всех контекстов
- **`context.WithCancel(parent)`** возвращает **новый** контекст и **функцию отмены**
- **`cancel()`** отменяет контекст. После этого `ctx.Done()` становится «закрытым каналом»
- **`defer cancel()`** — гарантия отмены при выходе из функции

`cancel` идемпотентен — можно вызывать несколько раз без вреда.

## Уровень 4: про горутину shutdown

```go
go func() {
    sig := <-sigChan
    log.Printf("Получен сигнал %v, начинаем graceful shutdown", sig)
    cancel()
}()
```

Это **отдельная горутина**, которая блокируется на чтении из `sigChan`. Когда придёт сигнал — она его прочитает, залогирует и вызовет `cancel()`. Это **разбудит** все, кто слушает `ctx.Done()`.

## Уровень 5: про Run с контекстом

```go
func (b *Bot) Run(ctx context.Context) {
    var offset int64 = 0
    for {
        // 1. Проверка отмены в начале каждой итерации
        select {
        case <-ctx.Done():
            log.Println("Контекст отменён, выходим из цикла")
            return
        default:
        }

        // 2. Запрос с контекстом
        updates, err := b.tg.GetUpdatesContext(ctx, offset)
        if err != nil {
            // Если ошибка из-за отмены — это нормально
            if ctx.Err() != nil {
                return
            }
            log.Printf("getUpdates error: %v", err)

            // 3. Retry с возможностью прерывания
            select {
            case <-ctx.Done():
                return
            case <-time.After(5 * time.Second):
            }
            continue
        }

        // 4. Обработка обновлений
        for _, upd := range updates {
            if upd.Message != nil && upd.Message.Text != "" {
                b.handleMessage(ctx, upd.Message)
            }
            if upd.UpdateID >= offset {
                offset = upd.UpdateID + 1
            }
        }
    }
}
```

Три места, где появляется контекст:

1. **Начало итерации** — проверка отмены через `select`
2. **HTTP-запрос** — `GetUpdatesContext(ctx, ...)`
3. **Retry-задержка** — `select + time.After` вместо `time.Sleep`

`select { case <-ctx.Done(): return; default: }` — это **non-blocking check**. Если контекст не отменён — `default` срабатывает мгновенно, идём дальше.

## Уровень 6: про HTTP-запрос с контекстом

Раньше:

```go
resp, err := c.client.Get(url)
```

Теперь:

```go
req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
if err != nil {
    return nil, fmt.Errorf("new request: %w", err)
}

resp, err := c.client.Do(req)
if err != nil {
    return nil, fmt.Errorf("do: %w", err)
}
defer resp.Body.Close()
```

Что изменилось:

- **`http.NewRequestWithContext`** создаёт `*http.Request` с привязкой к контексту
- **`client.Do(req)`** выполняет запрос. При отмене контекста запрос **прерывается**
- Это работает даже для **висящих** long polling запросов

Для POST с телом:

```go
req, err := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(data))
if err != nil { ... }
req.Header.Set("Content-Type", "application/json")

resp, err := c.client.Do(req)
```

Заголовки устанавливаются через `req.Header.Set` (не как параметр функции, как в `Post`).

## Уровень 7: про *Context методы в database/sql

```go
// Раньше
r.db.QueryRow(...)
r.db.Query(...)
r.db.Exec(...)

// Теперь
r.db.QueryRowContext(ctx, ...)
r.db.QueryContext(ctx, ...)
r.db.ExecContext(ctx, ...)
```

Сигнатура одинаковая, только первый параметр — контекст. PostgreSQL получает сигнал отмены и прерывает запрос.

Пример:

```go
func (r *Repository) AddContext(ctx context.Context, chatID int64, title string) (Task, error) {
    var t Task
    err := r.db.QueryRowContext(ctx, `
        INSERT INTO tasks (chat_id, user_task_id, title)
        VALUES ($1, COALESCE((SELECT MAX(user_task_id) FROM tasks WHERE chat_id = $1), 0) + 1, $2)
        RETURNING id, chat_id, user_task_id, title, done
    `, chatID, title).Scan(&t.ID, &t.ChatID, &t.UserTaskID, &t.Title, &t.Done)
    if err != nil {
        return Task{}, fmt.Errorf("insert: %w", err)
    }
    return t, nil
}
```

## Уровень 8: про проброс ctx через слои

```go
func (b *Bot) handleMessage(ctx context.Context, msg *Message) {
    // ...
    state := b.states.Get(chatID)
    if state != StateIdle {
        b.handleStateful(ctx, chatID, state, msg.Text)
        return
    }
    b.handleCommand(ctx, chatID, msg.Text)
}

func (b *Bot) handleAdd(ctx context.Context, chatID int64, arg string) {
    // ...
    task, err := b.repo.AddContext(ctx, chatID, arg)
    // ...
}
```

Контекст пробрасывается через **все** методы, которые могут выполнять I/O. Это много изменений в сигнатурах, но работа простая — добавить параметр и пробросить дальше.

## Уровень 9: про select с time.After

Раньше:

```go
time.Sleep(5 * time.Second)
```

Теперь:

```go
select {
case <-ctx.Done():
    return
case <-time.After(5 * time.Second):
}
```

`time.After(d)` возвращает канал, в который через `d` времени придёт значение. `select` ждёт **либо** этого, **либо** отмены контекста — что наступит раньше.

Это позволяет **прервать** ожидание при сигнале shutdown, а не ждать полные 5 секунд.

## Уровень 10: про exit code

После `defer db.Close()` и выхода из `main` без `os.Exit` — exit code = 0.

Если ты увидишь exit code 130 — это значит, что Go не успел перехватить SIGINT, и процесс был убит сигналом. Проверь:

- `signal.Notify` вызван **до** `bot.Run(ctx)`
- Канал с буфером 1
- Горутина запущена

Если exit code 1 — где-то `log.Fatal` или `os.Exit(1)`. Проверь логику.

## Если совсем тупик

Скажи Claude конкретно: «модуль 6 проекта 6, [проблема]». Самые частые ситуации:

- **«Бот не реагирует на Ctrl+C»** → не подписался на сигналы или не вызвал `cancel()` в горутине
- **«Бот завершается, но HTTP-запрос продолжает висеть»** → используешь `client.Get` вместо `client.Do(req)` с контекстом
- **«БД-запрос не отменяется»** → используешь `Query` вместо `QueryContext`
- **«Контекст не пробрасывается»** → проверь сигнатуры всех handle-функций
- **«time.Sleep вместо select»** → бот ждёт полные 5 секунд после Ctrl+C
