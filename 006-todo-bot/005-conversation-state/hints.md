# Модуль 5. Состояние диалога — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какой тип использовать для состояний? (Подсказка: что-то простое, что начинается с 0)
2. Где хранить состояния — в БД или в памяти?
3. Как сделать так, чтобы новый пользователь автоматически оказывался в `Idle`?
4. Как пользователь может выйти из состояния?
5. Что делать с `/cancel`?

## Уровень 2: про State через iota

```go
type State int

const (
    StateIdle State = iota  // 0
    StateWaitingForTaskTitle  // 1
)
```

`iota` — это специальный счётчик в Go, который начинается с 0 и инкрементируется в каждой строке `const` блока. Удобный способ объявить enum.

`StateIdle == 0` совпадает с **нулевым значением** для `int` — это значит, что для нового пользователя (без записи в map) `Get` вернёт `StateIdle` автоматически.

## Уровень 3: про StateStore

```go
type StateStore struct {
    mu     sync.Mutex
    states map[int64]State
}

func NewStateStore() *StateStore {
    return &StateStore{
        states: make(map[int64]State),
    }
}

func (s *StateStore) Get(chatID int64) State {
    s.mu.Lock()
    defer s.mu.Unlock()
    return s.states[chatID]  // вернёт 0 (StateIdle) для отсутствующего ключа
}

func (s *StateStore) Set(chatID int64, state State) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if state == StateIdle {
        delete(s.states, chatID)
    } else {
        s.states[chatID] = state
    }
}
```

Заметь: `Set(chatID, StateIdle)` **удаляет** запись. Это потому, что `Idle` — это «нет состояния», и хранить его в map бессмысленно.

## Уровень 4: про маршрутизацию в handleMessage

```go
func (b *Bot) handleMessage(msg *Message) {
    chatID := msg.Chat.ID
    text := msg.Text

    state := b.states.Get(chatID)
    if state != StateIdle {
        b.handleStateful(chatID, state, text)
        return
    }

    b.handleCommand(chatID, text)
}
```

Логика:
1. Сначала смотрим состояние
2. Если есть активное состояние (не Idle) — обработка по нему
3. Иначе — обычная команда

Заметь: **`return` после `handleStateful`** обязателен. Иначе после обработки состояния код **продолжит** выполнение и попадёт в `handleCommand`, что создаст двойную обработку.

## Уровень 5: про разделение handleMessage на части

Раньше у тебя был один `handleMessage`, который парсил команду и делал switch. Теперь надо разделить:

- **`handleMessage`** — маршрутизатор: state vs command
- **`handleCommand`** — старый код парсинга и switch
- **`handleStateful`** — новый код для обработки в состоянии

Это рефакторинг, не новая функциональность. Старый код переезжает в `handleCommand` без изменений (кроме добавления `/cancel`).

## Уровень 6: про handleStateful

```go
func (b *Bot) handleStateful(chatID int64, state State, text string) {
    text = strings.TrimSpace(text)

    // /cancel сбрасывает любое состояние
    if text == "/cancel" {
        b.states.Set(chatID, StateIdle)
        b.send(chatID, "Отменено.")
        return
    }

    // Любая команда — сброс и обработка
    if strings.HasPrefix(text, "/") {
        b.states.Set(chatID, StateIdle)
        b.send(chatID, "Состояние сброшено, обрабатываю команду.")
        b.handleCommand(chatID, text)
        return
    }

    // Обработка по состоянию
    switch state {
    case StateWaitingForTaskTitle:
        if text == "" {
            b.send(chatID, "Введи непустой текст или /cancel для отмены.")
            return
        }
        task, err := b.repo.Add(chatID, text)
        if err != nil {
            log.Printf("Add error: %v", err)
            b.send(chatID, "Ошибка при добавлении. Попробуй позже.")
            return
        }
        b.states.Set(chatID, StateIdle)
        b.send(chatID, fmt.Sprintf("Задача #%d добавлена: %s", task.UserTaskID, task.Title))

    default:
        log.Printf("Неизвестное состояние: %d для chat %d", state, chatID)
        b.states.Set(chatID, StateIdle)
        b.send(chatID, "Что-то пошло не так. Состояние сброшено.")
    }
}
```

Структура:
1. **`/cancel` всегда работает** (escape hatch)
2. **Любая `/команда` тоже работает** (дружественное поведение)
3. **Иначе** — обработка по конкретному состоянию через switch
4. **`default`** — для безопасности

## Уровень 7: про обновление handleAdd

Раньше:

```go
func (b *Bot) handleAdd(chatID int64, arg string) {
    arg = strings.TrimSpace(arg)
    if arg == "" {
        b.send(chatID, "Использование: /add <текст задачи>")
        return
    }
    // ... добавление ...
}
```

Теперь:

```go
func (b *Bot) handleAdd(chatID int64, arg string) {
    arg = strings.TrimSpace(arg)
    if arg == "" {
        b.states.Set(chatID, StateWaitingForTaskTitle)
        b.send(chatID, "Введи текст задачи следующим сообщением (или /cancel для отмены):")
        return
    }
    // ... добавление как раньше ...
}
```

Изменилась только **ветка с пустым arg** — вместо подсказки она устанавливает состояние.

## Уровень 8: про /cancel в handleCommand

Когда пользователь пишет `/cancel`, не находясь в состоянии, нужно как-то ответить. Иначе будет «Не понимаю команду» — что грубо.

```go
case "/cancel":
    b.send(chatID, "Нечего отменять.")
```

Это просто — добавить case в switch.

## Уровень 9: про порядок case в switch

В `handleCommand` порядок case не важен. В `handleStateful` тоже. Что важно — **проверки `/cancel` и `/команда` идут ДО switch**, потому что они применяются ко всем состояниям.

## Уровень 10: про разделение на файлы

Рекомендуемая структура:

```
practice-fsm/
├── go.mod
├── main.go
├── client.go         ← TelegramClient
├── types.go          ← Update, Message, Chat, User
├── repository.go     ← Repository (из модуля 4)
├── state.go          ← State, StateStore (новое в модуле 5)
├── bot.go            ← Bot, Run, handleMessage, handleStateful
├── handlers.go       ← handleStart, handleList, handleAdd, etc.
└── migrations/
    └── 001_create_tasks_table.sql
```

`state.go` — отдельный файл для FSM, чтобы не загромождать `bot.go`.

## Если совсем тупик

Скажи Claude конкретно: «модуль 5 проекта 6, [конкретная проблема]». Самые частые ситуации:

- **«Бот игнорирует состояние, обрабатывает текст как команду»** → не вызываешь `handleStateful` в `handleMessage`, или забыл `return` после
- **«Бот всегда в состоянии после /add»** → забыл `Set(StateIdle)` после успешного добавления
- **«/cancel не работает»** → проверь, что обработка `/cancel` идёт **до** switch по состоянию
- **«Состояние одного пользователя влияет на другого»** → ищи `chatID` во всех вызовах `Get`/`Set`
- **«panic: assignment to entry in nil map»** → забыл `make(map...)` в `NewStateStore`
