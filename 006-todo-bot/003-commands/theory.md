# Модуль 3. Команды — теория

## Контекст

В модуле 2 твой бот эхом отвечает на любое сообщение. Это работает, но **не полезно**. В этом модуле ты добавишь **команды** — структурированный способ взаимодействия, к которому привыкли пользователи Telegram:

- `/start` — приветствие
- `/help` — справка
- `/list` — показать задачи
- `/add <текст>` — добавить
- `/done <id>` — отметить выполненной
- `/delete <id>` — удалить

К концу модуля 3 у тебя будет **функционально полный** to-do бот. Без БД (это модуль 4) — данные пока в памяти, в `map[chatID][]Task`. Но логически — всё работает.

## Ядро 1: что такое команды в Telegram

С точки зрения Telegram, **команда** — это просто текстовое сообщение, начинающееся с `/`. Никакой особой обработки на стороне Telegram нет. Это просто соглашение для удобства пользователей.

Telegram, однако, делает несколько приятных вещей с командами:

- Команды **подсвечиваются** синим в чате (как ссылки)
- При вводе `/` появляется **выпадающий список** команд бота (которые ты установил через `/setcommands` в @BotFather)
- При нажатии на команду в списке — она автоматически отправляется боту

С точки зрения **твоего кода** — это просто `msg.Text`, начинающийся с `/`. Парсить и обрабатывать ты должен **сам**.

## Ядро 2: парсинг команды

Сообщение `/add Купить молоко` нужно разбить на **команду** (`/add`) и **аргумент** (`Купить молоко`).

Вспомни Проект 2 (REPL): ты делал то же самое через `strings.SplitN(line, " ", 2)`. Здесь то же:

```go
parts := strings.SplitN(text, " ", 2)
command := parts[0]  // "/add"
var arg string
if len(parts) > 1 {
    arg = parts[1]   // "Купить молоко"
}
```

Несколько особенностей Telegram, которые надо учесть:

### 1. Команда может содержать имя бота

Если бот в группе, пользователь может написать `/add@lev_todo_bot Купить молоко` — это вариант адресации команды конкретному боту. Тебе нужно это обработать:

```go
command := parts[0]
if idx := strings.Index(command, "@"); idx != -1 {
    command = command[:idx]  // обрезаем "@bot_name"
}
```

После этой строки `command` всегда вида `/add` или `/list`, без `@`.

### 2. Команда чувствительна к регистру

`/add` и `/Add` — это **разные** команды. Стандартное соглашение — все команды в нижнем регистре. Ты можешь дополнительно нормализовать через `strings.ToLower(command)`.

### 3. Аргумент может быть пустым

`/add` без аргумента — это валидное сообщение. Обработай его:

```go
if command == "/add" && arg == "" {
    bot.SendMessage(chat, "Использование: /add <текст задачи>")
    return
}
```

В модуле 5 (FSM) ты улучшишь это — бот будет **запоминать**, что ждёт текст задачи, и интерпретировать **следующее** сообщение как title.

## Ядро 3: switch по командам

Стандартный паттерн обработки:

```go
func (b *Bot) handleMessage(msg *Message) {
    text := msg.Text
    chatID := msg.Chat.ID

    parts := strings.SplitN(text, " ", 2)
    command := strings.ToLower(parts[0])
    if idx := strings.Index(command, "@"); idx != -1 {
        command = command[:idx]
    }
    var arg string
    if len(parts) > 1 {
        arg = parts[1]
    }

    switch command {
    case "/start":
        b.handleStart(chatID)
    case "/help":
        b.handleHelp(chatID)
    case "/list":
        b.handleList(chatID)
    case "/add":
        b.handleAdd(chatID, arg)
    case "/done":
        b.handleDone(chatID, arg)
    case "/delete":
        b.handleDelete(chatID, arg)
    default:
        b.handleUnknown(chatID)
    }
}
```

Это — точно та же структура, что в REPL из Проекта 2. Принцип «чтение → парсинг → диспетчеризация → обработка» одинаков для CLI, для HTTP-сервера и для бота. Только источник входа разный.

## Ядро 4: per-user storage

**Это самая важная мысль модуля 3.**

В Проектах 2-5 у тебя был **один общий список задач** — для одного пользователя (тебя). В боте такой подход не работает: **разные пользователи должны иметь разные данные**. Если Лев и Аня пишут одному и тому же боту, они должны видеть **разные** списки.

Telegram даёт каждому пользователю уникальный `chat_id` (для личных чатов он совпадает с `user_id`). Это — **естественный ключ** для разделения данных.

Простейшая структура в памяти:

```go
type Storage struct {
    mu        sync.Mutex
    tasksByChat map[int64][]Task
    nextIDByChat map[int64]int64
}

func NewStorage() *Storage {
    return &Storage{
        tasksByChat:  make(map[int64][]Task),
        nextIDByChat: make(map[int64]int64),
    }
}
```

Заметь:

1. **`map[int64][]Task`** — у каждого `chat_id` свой слайс задач
2. **`nextIDByChat`** — у каждого пользователя свои id (1, 2, 3...). Это удобнее, чем глобальный `nextID`, потому что пользователи видят понятные номера у себя в списке
3. **`sync.Mutex`** — потому что бот **одновременно** может обрабатывать сообщения от разных пользователей. В Telegram update-ы от одного пользователя приходят последовательно, но между разными пользователями — параллельно.

⚠️ **Тонкость с горутинами:** в нашем модуле 3 цикл обработки **синхронный** — мы обрабатываем сообщения по одному, и race condition не возникает естественным путём. Но **хорошая привычка** — защищать общее состояние мьютексом сразу. В реальных ботах часто используют горутины для параллельной обработки.

## Ядро 5: методы Storage

```go
func (s *Storage) Add(chatID int64, title string) Task {
    s.mu.Lock()
    defer s.mu.Unlock()

    s.nextIDByChat[chatID]++
    task := Task{
        ID:    s.nextIDByChat[chatID],
        Title: title,
    }
    s.tasksByChat[chatID] = append(s.tasksByChat[chatID], task)
    return task
}

func (s *Storage) List(chatID int64) []Task {
    s.mu.Lock()
    defer s.mu.Unlock()

    tasks := s.tasksByChat[chatID]
    result := make([]Task, len(tasks))
    copy(result, tasks)
    return result
}

func (s *Storage) Done(chatID int64, id int64) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    tasks := s.tasksByChat[chatID]
    for i := range tasks {
        if tasks[i].ID == id {
            tasks[i].Done = true
            return nil
        }
    }
    return fmt.Errorf("задача не найдена: %d", id)
}

func (s *Storage) Delete(chatID int64, id int64) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    tasks := s.tasksByChat[chatID]
    for i, t := range tasks {
        if t.ID == id {
            s.tasksByChat[chatID] = append(tasks[:i], tasks[i+1:]...)
            return nil
        }
    }
    return fmt.Errorf("задача не найдена: %d", id)
}
```

Заметь:

1. **Каждый метод фильтрует по `chatID`** — пользователи изолированы
2. **`List` возвращает копию** (как в Проекте 4 модуле 5) — защита от внешних изменений
3. **`nextIDByChat[chatID]++`** — счётчик per-user. Если у пользователя нет записи в map, `int64` по умолчанию `0`, инкремент даёт 1. Map в Go возвращает нулевое значение для отсутствующего ключа.

## Ядро 6: handle-функции

```go
func (b *Bot) handleStart(chatID int64) {
    text := "Привет! Я твой to-do бот. Команды:\n" +
        "/list — показать задачи\n" +
        "/add <текст> — добавить задачу\n" +
        "/done <id> — отметить выполненной\n" +
        "/delete <id> — удалить\n" +
        "/help — справка"
    b.send(chatID, text)
}

func (b *Bot) handleHelp(chatID int64) {
    b.handleStart(chatID)
}

func (b *Bot) handleList(chatID int64) {
    tasks := b.storage.List(chatID)
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
        fmt.Fprintf(&sb, "%d. [%s] %s\n", t.ID, check, t.Title)
    }
    b.send(chatID, sb.String())
}

func (b *Bot) handleAdd(chatID int64, arg string) {
    arg = strings.TrimSpace(arg)
    if arg == "" {
        b.send(chatID, "Использование: /add <текст задачи>")
        return
    }
    task := b.storage.Add(chatID, arg)
    b.send(chatID, fmt.Sprintf("Задача #%d добавлена: %s", task.ID, task.Title))
}

func (b *Bot) handleDone(chatID int64, arg string) {
    id, err := parseTaskID(arg)
    if err != nil {
        b.send(chatID, "Использование: /done <id>")
        return
    }
    if err := b.storage.Done(chatID, id); err != nil {
        b.send(chatID, err.Error())
        return
    }
    b.send(chatID, fmt.Sprintf("Задача #%d отмечена выполненной", id))
}

func (b *Bot) handleDelete(chatID int64, arg string) {
    id, err := parseTaskID(arg)
    if err != nil {
        b.send(chatID, "Использование: /delete <id>")
        return
    }
    if err := b.storage.Delete(chatID, id); err != nil {
        b.send(chatID, err.Error())
        return
    }
    b.send(chatID, fmt.Sprintf("Задача #%d удалена", id))
}

func (b *Bot) handleUnknown(chatID int64) {
    b.send(chatID, "Не понимаю команду. Используй /help для справки.")
}

func parseTaskID(arg string) (int64, error) {
    arg = strings.TrimSpace(arg)
    if arg == "" {
        return 0, fmt.Errorf("пустой id")
    }
    return strconv.ParseInt(arg, 10, 64)
}

func (b *Bot) send(chatID int64, text string) {
    if err := b.tg.SendMessage(chatID, text); err != nil {
        log.Printf("send error: %v", err)
    }
}
```

Несколько новых вещей:

1. **`strings.Builder`** — эффективный способ собрать большую строку из частей. Используется через `fmt.Fprintf(&sb, ...)`. Альтернативы — конкатенация `+=` (медленно для большого текста) или `strings.Join` (нужен слайс).

2. **`b.send(...)` helper** — обёртка над `bot.SendMessage`, которая логирует ошибки. Не возвращает ошибку — мы и так ничего полезного с ней не сделаем в обработчике.

3. **`parseTaskID` helper** — выносим парсинг id в отдельную функцию, чтобы не дублировать в `handleDone` и `handleDelete`.

## Ядро 7: тип Bot — собирающая структура

Чтобы не передавать `tg`, `storage` отдельно в каждую handle-функцию, оборачиваем их в один тип:

```go
type Bot struct {
    tg      *TelegramClient
    storage *Storage
}

func NewBot(tg *TelegramClient, storage *Storage) *Bot {
    return &Bot{tg: tg, storage: storage}
}
```

И главный цикл:

```go
func (b *Bot) Run() {
    me, err := b.tg.GetMe()
    if err != nil {
        log.Fatalf("getMe: %v", err)
    }
    log.Printf("Бот запущен: @%s", me.Username)

    var offset int64 = 0
    for {
        updates, err := b.tg.GetUpdates(offset)
        if err != nil {
            log.Printf("getUpdates error: %v", err)
            time.Sleep(5 * time.Second)
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

И `main`:

```go
func main() {
    token := os.Getenv("BOT_TOKEN")
    if token == "" {
        log.Fatal("BOT_TOKEN не задан")
    }

    tg := NewTelegramClient(token)
    storage := NewStorage()
    bot := NewBot(tg, storage)
    bot.Run()
}
```

Заметь, как чисто получается: `main` только инициализирует, `Bot.Run()` крутит цикл, `handleMessage` диспетчеризует, отдельные `handle*` функции делают свою работу. Это **разделение слоёв**, которое ты уже умеешь делать.

## Ядро 8: что осталось плохо

В этом модуле всё работает, но есть две проблемы, которые ты решишь дальше:

### Проблема 1: Данные в памяти

Перезапустил бота — все задачи пропали. Это решает **модуль 4** (PostgreSQL).

### Проблема 2: `/add` без аргумента — неудобно

Пользователю приходится набирать `/add Купить молоко` одной строкой. Естественнее было бы:

```
Ты:    /add
Бот:   Введи текст задачи следующим сообщением:
Ты:    Купить молоко
Бот:   Задача #1 добавлена
```

Это требует **запоминать состояние** — «бот ждёт от этого пользователя текст задачи». Это **FSM**, и она в **модуле 5**.

## Вопросы на понимание

1. Что такое команда в Telegram с точки зрения Telegram?
2. Как разбить `/add Купить молоко` на команду и аргумент?
3. Что делать с `@bot_name` после команды в групповых чатах?
4. Зачем нужен **per-user** storage? Что произойдёт, если использовать общий список?
5. Почему `nextID` тоже **per-user**, а не глобальный?
6. Зачем `sync.Mutex` в `Storage`, если мы обрабатываем сообщения по одному?
7. Что такое `strings.Builder` и в чём его польза?
8. Почему `List` возвращает **копию** слайса, а не оригинал?
9. Какие две проблемы остаются после модуля 3, и в каких модулях они решаются?
