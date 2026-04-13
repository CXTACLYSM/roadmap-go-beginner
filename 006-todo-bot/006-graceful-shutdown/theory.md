# Модуль 6. Graceful shutdown — теория

## Контекст

Это **финальный** модуль Проекта 6. Твой бот функционально работает: команды, БД, состояния. Но есть одна вещь, которая отличает «учебный пример» от «production-ready сервиса» — **корректное завершение**.

Сейчас ты останавливаешь бота через **Ctrl+C**, и он умирает «грубо»: соединения с БД и Telegram остаются «висеть», текущие запросы прерываются, состояние в памяти теряется. Для учебных целей это OK. Для production — категорически нет.

В этом модуле ты:

- Узнаешь, что такое **сигналы** в Unix
- Поймёшь, **зачем** нужен `context.Context`
- Реализуешь **graceful shutdown**: при сигнале остановки бот доделает текущую обработку, закроет ресурсы и завершится корректно
- Прокинешь контекст через все слои: `Run` → `GetUpdates` → HTTP-запросы → БД-запросы

После этого модуля твой бот станет **production-ready**.

## Ядро 1: что такое сигналы

**Сигнал** — это асинхронное сообщение, которое операционная система может послать процессу. Их около 30 разных. Самые важные:

| Сигнал | Что значит | Можно перехватить? |
|--------|------------|---------------------|
| `SIGINT` | Прерывание (Ctrl+C) | Да |
| `SIGTERM` | Запрос на завершение (от systemd, k8s) | Да |
| `SIGKILL` | Принудительное убийство (`kill -9`) | **Нет** |
| `SIGHUP` | Терминал закрыт | Да |
| `SIGUSR1`, `SIGUSR2` | Пользовательские | Да |

`SIGKILL` нельзя перехватить — это «выключатель» из розетки. Все остальные можно.

В обычной работе ты остановишь бота через `Ctrl+C` (SIGINT) или kubernetes пошлёт `SIGTERM` при перезапуске пода. **Оба эти сигнала ожидают**, что приложение завершится **корректно** в течение разумного времени.

## Ядро 2: signal.Notify в Go

Стандартная библиотека Go даёт пакет `os/signal` для работы с сигналами:

```go
import (
    "os"
    "os/signal"
    "syscall"
)

sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)

// ждём сигнал
sig := <-sigChan
log.Printf("Получен сигнал: %v", sig)
```

Разберём:

1. **`make(chan os.Signal, 1)`** — буферизированный канал размером 1. Буфер важен: если сигнал придёт **до** того, как мы начали читать, он бы «пропал». С буфером — будет ждать.

2. **`signal.Notify(sigChan, ...)`** — регистрирует канал как получателя указанных сигналов. Когда придёт сигнал, Go запишет его в канал.

3. **`os.Interrupt`** — это `SIGINT` (Ctrl+C), но кросс-платформенно (на Windows работает аналогично).

4. **`syscall.SIGTERM`** — сигнал от systemd/kubernetes/docker stop.

5. **`<-sigChan`** — блокирующее чтение из канала. Программа стоит здесь, пока не придёт сигнал.

## Ядро 3: что такое context.Context

`context.Context` — это **стандартный механизм** Go для:

- **Отмены** длительных операций (например, при сигнале shutdown)
- **Таймаутов** (например, «выполни запрос за 30 секунд или сдавайся»)
- **Передачи request-scoped данных** между функциями (например, request_id для трейсинга)

В этом модуле нас интересуют **первые две** возможности.

Главная идея: создать **корневой контекст**, который **отменяется** при сигнале. Передать его во **все** длительные операции. Когда контекст отменён — операции тоже отменяются.

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// где-то в горутине:
go func() {
    <-sigChan
    log.Println("Сигнал получен, отменяем контекст")
    cancel()
}()

// другие функции получают ctx и слушают ctx.Done()
bot.Run(ctx)
```

`context.Background()` — это **пустой** root-контекст без отмены и таймаута. Все контексты в Go начинаются от него.

`context.WithCancel(parent)` создаёт **новый** контекст, наследник `parent`, который можно отменить через возвращаемую функцию `cancel`.

`cancel()` — отменяет контекст. После этого `ctx.Done()` (канал) становится «закрытым», и все, кто его слушают, узнают.

`defer cancel()` — гарантия, что контекст отменится при выходе из функции (даже если сигнала нет, например, из-за паники).

## Ядро 4: использование контекста в Run

Раньше:

```go
func (b *Bot) Run() {
    var offset int64 = 0
    for {
        updates, err := b.tg.GetUpdates(offset)
        // ...
    }
}
```

Теперь:

```go
func (b *Bot) Run(ctx context.Context) {
    var offset int64 = 0
    for {
        select {
        case <-ctx.Done():
            log.Println("Контекст отменён, выходим из цикла")
            return
        default:
        }

        updates, err := b.tg.GetUpdatesContext(ctx, offset)
        if err != nil {
            if ctx.Err() != nil {
                return  // отмена через контекст — это не ошибка
            }
            log.Printf("getUpdates error: %v", err)
            select {
            case <-ctx.Done():
                return
            case <-time.After(5 * time.Second):
            }
            continue
        }

        for _, upd := range updates {
            if upd.Message != nil && upd.Message.Text != "" {
                b.handleMessage(upd.Message)
            }
            if upd.UpdateID >= offset {
                offset = upd.UpdateID + 1
            }
        }
    }
}
```

Несколько важных моментов:

1. **`ctx context.Context` — первый аргумент**. Это конвенция Go: контекст всегда первый.

2. **`select { case <-ctx.Done(): return; default: }`** — неблокирующая проверка отмены в начале каждой итерации. Если контекст отменён — выходим.

3. **`GetUpdatesContext(ctx, offset)`** — версия `GetUpdates` с контекстом. Передаёт контекст в HTTP-запрос.

4. **Проверка `ctx.Err()` после ошибки** — если контекст отменён, ошибка от `GetUpdates` может быть «context canceled». Это не настоящая ошибка, не нужно её логировать как баг.

5. **`select` с `time.After` вместо `time.Sleep`** — позволяет **прервать ожидание** при отмене контекста. С `time.Sleep` бот бы спал 5 секунд независимо от сигнала.

## Ядро 5: HTTP-запросы с контекстом

Чтобы `client.Get(url)` мог быть отменён через контекст, нужно использовать `http.NewRequestWithContext`:

```go
func (c *TelegramClient) GetUpdatesContext(ctx context.Context, offset int64) ([]Update, error) {
    url := fmt.Sprintf("%s/getUpdates?offset=%d&timeout=30", c.apiURL, offset)

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("new request: %w", err)
    }

    resp, err := c.client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("do: %w", err)
    }
    defer resp.Body.Close()

    // ... парсинг как раньше ...
}
```

Что происходит:

1. **`http.NewRequestWithContext(ctx, ...)`** создаёт запрос, привязанный к контексту
2. **`client.Do(req)`** выполняет запрос
3. Если контекст отменяется во время `Do` — запрос **прерывается**, и `Do` возвращает ошибку «context canceled»

Это — **ключевая способность**. Без неё `GetUpdates` мог бы висеть 30 секунд (long polling timeout), даже если ты уже хочешь shutdown. С контекстом — отмена мгновенная.

## Ядро 6: контекст для БД

`database/sql` тоже поддерживает контексты через **`*Context`** методы:

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

Заметь: `QueryRowContext` вместо `QueryRow`. Аналогично — `QueryContext`, `ExecContext`.

При отмене контекста БД-запрос тоже отменяется (PostgreSQL получит сигнал отмены и откатит транзакцию, если она была).

**Идиома Go**: если у тебя есть **обе** версии (`Query` и `QueryContext`) — **всегда** используй ту, что с контекстом. Версии без контекста — для скриптов и учебных примеров.

## Ядро 7: проброс контекста через handleMessage

```go
func (b *Bot) handleMessage(ctx context.Context, msg *Message) {
    // ...
    if state != StateIdle {
        b.handleStateful(ctx, chatID, state, text)
        return
    }
    b.handleCommand(ctx, chatID, text)
}

func (b *Bot) handleAdd(ctx context.Context, chatID int64, arg string) {
    // ...
    task, err := b.repo.AddContext(ctx, chatID, arg)
    // ...
}
```

**Контекст пробрасывается во все методы**, которые могут делать длительные операции. Это идиома Go: **первый параметр — контекст**.

Зачем? Потому что:

- Если shutdown пришёл, пока бот вставляет задачу в БД — мы хотим **отменить** этот INSERT, а не ждать
- Без контекста — нужно ждать завершения, что может занять долго при тормозящей БД

Это правило: **контекст в каждый метод, который может делать I/O или другие длительные операции**.

## Ядро 8: shutdown handler

Главная функция, которая собирает всё вместе:

```go
func main() {
    token := os.Getenv("BOT_TOKEN")
    dsn := os.Getenv("DATABASE_URL")
    if token == "" || dsn == "" {
        log.Fatal("BOT_TOKEN и DATABASE_URL обязательны")
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

    tg := NewTelegramClient(token)
    repo := NewRepository(db)
    states := NewStateStore()
    bot := NewBot(tg, repo, states)

    // Контекст для всего приложения
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Перехват сигналов
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)

    // Горутина для shutdown
    go func() {
        sig := <-sigChan
        log.Printf("Получен сигнал %v, начинаем graceful shutdown", sig)
        cancel()
    }()

    log.Println("Бот запускается...")
    bot.Run(ctx)
    log.Println("Бот остановлен корректно")
}
```

Структура `main`:

1. Конфиг и зависимости (как раньше)
2. **Контекст с отменой** — корневой для всего приложения
3. **`defer cancel()`** — гарантия отмены при выходе
4. **Канал для сигналов** + `signal.Notify`
5. **Горутина**, которая ждёт сигнал и вызывает `cancel()`
6. **`bot.Run(ctx)`** — блокирующий вызов до отмены контекста
7. После выхода из `Run` — лог «остановлен корректно»

## Ядро 9: что происходит при Ctrl+C

Полный таймлайн:

1. Ты нажимаешь Ctrl+C в терминале
2. Терминал посылает `SIGINT` процессу бота
3. Go перехватывает сигнал, кладёт в `sigChan`
4. Горутина читает из `sigChan`, логирует «получен сигнал»
5. Горутина вызывает `cancel()`
6. `ctx.Done()` становится «закрытым»
7. `bot.Run(ctx)` в следующей итерации видит `ctx.Done()` → выходит из цикла
8. Если в этот момент висит `GetUpdatesContext` — он тоже отменяется через контекст HTTP-запроса
9. `Run` возвращает управление в `main`
10. `defer db.Close()` срабатывает — закрывается пул соединений
11. `defer cancel()` срабатывает (уже после `cancel`, но идемпотентно)
12. `main` завершается, программа выходит с кодом 0

Сравни с «грубым» завершением в модулях 1-5:

1. Ctrl+C
2. Терминал посылает SIGINT
3. Программа **умирает** мгновенно
4. Активный HTTP-запрос обрывается (Telegram может отдать ответ, но мы его не получим)
5. Если шла транзакция в БД — она будет **откатана** PostgreSQL по таймауту, но не сразу
6. Соединения с БД остаются «болтаться» какое-то время

Graceful shutdown — это **дисциплина**, а не магия. Просто говорим себе: «давайте закроем то, что открыли».

## Ядро 10: shutdown timeout

В реальных системах добавляют **таймаут на shutdown**: если за 30 секунд бот не завершился — убить принудительно. Это защищает от зависших операций.

```go
go func() {
    sig := <-sigChan
    log.Printf("Получен сигнал %v, начинаем shutdown", sig)
    cancel()

    // Если за 30 секунд не вышли — принудительно
    select {
    case sig := <-sigChan:
        log.Printf("Второй сигнал %v, принудительный выход", sig)
        os.Exit(1)
    case <-time.After(30 * time.Second):
        log.Println("Timeout, принудительный выход")
        os.Exit(1)
    }
}()
```

Это — production-grade паттерн. Для нашего модуля 6 — упрощённая версия без таймаута, но **знать про неё обязательно**.

## Вопросы на понимание

1. Что такое **сигнал** в Unix? Какие два самых важных для shutdown?
2. Можно ли перехватить `SIGKILL`?
3. Что такое `context.Context` и какие три задачи он решает?
4. Зачем `make(chan os.Signal, 1)` с буфером 1?
5. Что делает `context.WithCancel`?
6. Почему контекст должен быть **первым** параметром функции?
7. Чем `client.Do(req)` отличается от `client.Get(url)` с точки зрения контекста?
8. Что произойдёт, если БД-запрос идёт во время отмены контекста?
9. Почему мы используем `select + time.After` вместо `time.Sleep` в retry-логике?
10. Какие шаги graceful shutdown происходят при Ctrl+C?
