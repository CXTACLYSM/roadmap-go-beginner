# Модуль 3. Команды — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какой паттерн ты использовал для парсинга команд в Проекте 2 (REPL)?
2. Как разбить строку «/add Купить молоко» на команду и аргумент?
3. Какой ключ нужен в map для разделения данных пользователей?
4. Что вернёт `m[key]`, если ключа нет в map?

## Уровень 2: про парсинг команды

Шаблон такой же, что в Проекте 2:

```go
parts := strings.SplitN(text, " ", 2)
command := parts[0]
var arg string
if len(parts) > 1 {
    arg = parts[1]
}
```

Дополнительно — обрезка `@bot_name`:

```go
if idx := strings.Index(command, "@"); idx != -1 {
    command = command[:idx]
}
```

И опционально — нижний регистр:

```go
command = strings.ToLower(command)
```

## Уровень 3: про map с дефолтными значениями

В Go обращение к map по несуществующему ключу **не падает**, а возвращает **нулевое значение** типа:

```go
var nextIDByChat map[int64]int64 = make(map[int64]int64)

id := nextIDByChat[12345]  // 0 (нулевое для int64)
nextIDByChat[12345]++       // теперь 1
```

Это удобно для счётчиков: первое обращение даёт 0, инкремент даёт 1.

То же для слайсов:

```go
var tasksByChat map[int64][]Task = make(map[int64][]Task)

tasks := tasksByChat[12345]  // nil (нулевое для []Task)
tasksByChat[12345] = append(tasks, Task{})  // добавляет в nil-слайс — это ОК
```

`append` к `nil`-слайсу работает корректно — возвращает новый слайс с одним элементом.

## Уровень 4: про методы Storage

Шаблон:

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
```

Несколько моментов:

- **`s.nextIDByChat[chatID]++`** — инкремент. Если ключа не было — теперь стало 1
- **`s.tasksByChat[chatID] = append(...)`** — присваиваем обратно (как обычно с append)
- **Lock + defer Unlock** — защита от параллельных запросов

## Уровень 5: про возврат копии из List

Та же тонкость, что в Проекте 4 модуле 5:

```go
func (s *Storage) List(chatID int64) []Task {
    s.mu.Lock()
    defer s.mu.Unlock()

    tasks := s.tasksByChat[chatID]
    result := make([]Task, len(tasks))
    copy(result, tasks)
    return result
}
```

Если вернуть `tasks` напрямую — снаружи Lock-а слайс может измениться параллельно с другим запросом. Возврат копии изолирует.

## Уровень 6: про Done и Delete

Поиск + изменение через индекс (помнишь Проект 2?):

```go
func (s *Storage) Done(chatID int64, id int64) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    tasks := s.tasksByChat[chatID]
    for i := range tasks {
        if tasks[i].ID == id {
            tasks[i].Done = true   // ← через индекс, не через range-копию
            return nil
        }
    }
    return fmt.Errorf("задача не найдена: %d", id)
}
```

Удаление — через `append + slicing`:

```go
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

Это всё — повторение того, что ты делал в Проекте 2 модуле 4.

## Уровень 7: про strings.Builder

Для сборки списка задач в одно сообщение:

```go
var sb strings.Builder
for _, t := range tasks {
    check := " "
    if t.Done {
        check = "x"
    }
    fmt.Fprintf(&sb, "%d. [%s] %s\n", t.ID, check, t.Title)
}
text := sb.String()
```

`strings.Builder` — это эффективный способ собрать большую строку. В отличие от `+=` он не аллоцирует новую строку на каждой итерации.

`fmt.Fprintf(&sb, ...)` работает с любым `io.Writer`, и `strings.Builder` реализует этот интерфейс.

## Уровень 8: про parseTaskID

```go
func parseTaskID(arg string) (int64, error) {
    arg = strings.TrimSpace(arg)
    if arg == "" {
        return 0, fmt.Errorf("пустой id")
    }
    return strconv.ParseInt(arg, 10, 64)
}
```

`ParseInt(s, base, bitSize)`:
- `s` — строка
- `base` — система счисления (10 для десятичной)
- `bitSize` — размер целого: 8, 16, 32, 64

Используем 64, потому что у нас `int64` для id (как в проекте 5 для совместимости с `BIGSERIAL`).

## Уровень 9: про helper send

```go
func (b *Bot) send(chatID int64, text string) {
    if err := b.tg.SendMessage(chatID, text); err != nil {
        log.Printf("send error to chat %d: %v", chatID, err)
    }
}
```

Эта обёртка убирает дублирование `if err != nil { log... }` в каждом обработчике. Помни: ошибка отправки не должна валить бота.

## Уровень 10: про разделение на файлы

Для модуля 3 рекомендую такую структуру:

```
practice-commands/
├── go.mod
├── main.go         ← инициализация
├── client.go       ← TelegramClient (из модулей 1-2)
├── types.go        ← Update, Message, Chat, User
├── task.go         ← Task
├── storage.go      ← Storage
├── bot.go          ← Bot, Run, handleMessage
└── handlers.go     ← handleStart, handleList, etc.
```

Все в `package main`. Дисциплина одного файла = одна ответственность.

## Если совсем тупик

Скажи Claude конкретно: «модуль 3 проекта 6, [конкретная проблема]». Самые частые ситуации:

- **«Команды не распознаются»** → проверь, делаешь ли ты `strings.ToLower` (если пользователь пишет `/Add`)
- **«Один пользователь видит задачи другого»** → ищи использование `chatID` во **всех** методах Storage
- **«panic: assignment to entry in nil map»** → забыл `make(map...)` в конструкторе
- **«Done не сохраняется»** → используешь range-копию вместо `tasks[i].Done = true`
- **«Telegram error 400: message is too long»** → список длиннее 4096 символов, нужна разбивка
