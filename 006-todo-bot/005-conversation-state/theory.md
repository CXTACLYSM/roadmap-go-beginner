# Модуль 5. Состояние диалога — теория

## Контекст

Сейчас твой бот работает как «командная строка»: пользователь вводит `/add Купить молоко` одной строкой, бот выполняет, всё. Это работает, но **неудобно**. Естественнее было бы:

```
Ты:    /add
Бот:   Введи текст задачи следующим сообщением:
Ты:    Купить молоко
Бот:   Задача #1 добавлена: Купить молоко
```

Заметь: между сообщением `/add` и сообщением `Купить молоко` бот **что-то помнит**. А именно — что от **этого конкретного пользователя** он ждёт текст. Это и есть **состояние диалога**.

В этом модуле ты добавишь простейшую **state machine** для управления такими сценариями.

## Ядро 1: что такое state machine

**Finite State Machine (FSM)** — это формальная модель, в которой:

- Есть **множество состояний** (например, `Idle`, `WaitingForTaskTitle`, `WaitingForConfirmation`)
- Есть **переходы** между состояниями, вызванные событиями (входящими сообщениями)
- В каждый момент времени система находится **ровно в одном** состоянии
- Действия зависят от **текущего состояния** и **входящего события**

Для нашего бота:

```
Состояния:
  Idle                  — обычный режим, ждём команду
  WaitingForTaskTitle   — ждём от пользователя текст для добавления задачи

Переходы:
  Idle + "/add" (пустое тело)  → WaitingForTaskTitle
  Idle + "/add Купить молоко"  → Idle (задача добавлена сразу)
  WaitingForTaskTitle + "Купить молоко" → Idle (задача добавлена)
  WaitingForTaskTitle + "/cancel" → Idle (отменили без действия)
```

Это очень простая FSM — всего два состояния. В реальных ботах их могут быть десятки (регистрация, опросы, многошаговые формы).

## Ядро 2: где хранить состояние

**Состояние — per user.** У каждого `chat_id` своё текущее состояние, которое не пересекается с другими.

Самый простой способ — `map[int64]State`:

```go
type State int

const (
    StateIdle State = iota
    StateWaitingForTaskTitle
)

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
    return s.states[chatID]  // вернёт StateIdle (0) для нового пользователя
}

func (s *StateStore) Set(chatID int64, state State) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if state == StateIdle {
        delete(s.states, chatID)  // экономия памяти: не храним idle
    } else {
        s.states[chatID] = state
    }
}
```

Несколько важных моментов:

1. **`State` — это `int`** через `iota`. `StateIdle == 0`, `StateWaitingForTaskTitle == 1`. Простое перечисление.

2. **`StateIdle` — это `0`**, что совпадает с **нулевым значением** для `int`. Это значит: для любого нового пользователя, у которого нет записи в map, `Get` вернёт `StateIdle`. Бесплатная инициализация.

3. **`Set` удаляет запись для `StateIdle`** — экономия памяти. Зачем хранить «у этого пользователя нет состояния»?

4. **`sync.Mutex`** — потому что разные сообщения от разных пользователей могут обрабатываться параллельно.

5. **Состояние НЕ хранится в БД.** Только в памяти. Это значит, что после перезапуска бота **состояние теряется**. Для нашего проекта это нормально — пользователь просто увидит, что бот «забыл» и начнёт сначала. Для production-ботов с длинными многошаговыми формами состояние хранят в Redis или БД.

## Ядро 3: маршрутизация — состояние или команда

Главное изменение в `handleMessage`:

```go
func (b *Bot) handleMessage(msg *Message) {
    chatID := msg.Chat.ID
    text := msg.Text

    // 1. Сначала — есть ли активное состояние?
    state := b.states.Get(chatID)
    if state != StateIdle {
        b.handleStateful(chatID, state, text)
        return
    }

    // 2. Иначе — обрабатываем как команду
    b.handleCommand(chatID, text)
}
```

Логика:

- **Если у пользователя есть активное состояние** (не `Idle`) — следующее сообщение интерпретируется в контексте этого состояния, **не как команда**
- **Иначе** — обычная обработка команды через switch

Это как «модальное окно» в десктопных приложениях: пока модал открыт, остальные кнопки игнорируются.

## Ядро 4: handleStateful — обработка по состоянию

```go
func (b *Bot) handleStateful(chatID int64, state State, text string) {
    text = strings.TrimSpace(text)

    // Команда /cancel сбрасывает любое состояние
    if text == "/cancel" {
        b.states.Set(chatID, StateIdle)
        b.send(chatID, "Отменено.")
        return
    }

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
        log.Printf("Неизвестное состояние: %d", state)
        b.states.Set(chatID, StateIdle)
        b.send(chatID, "Что-то пошло не так. Состояние сброшено.")
    }
}
```

Несколько важных моментов:

1. **`/cancel` обрабатывается всегда** — это **escape hatch**, который позволяет пользователю выйти из любого состояния. **Каждая FSM должна иметь способ отмены**, иначе пользователь застрянет.

2. **Switch по состоянию** — для каждого состояния своя логика обработки.

3. **После успешной обработки — `Set(chatID, StateIdle)`** — переход обратно в обычный режим.

4. **`default` ветка** — на случай неизвестного состояния (например, после кода-апдейта). Безопасный сброс.

## Ядро 5: обновление handleAdd

```go
func (b *Bot) handleAdd(chatID int64, arg string) {
    arg = strings.TrimSpace(arg)
    if arg == "" {
        // Аргумента нет — переходим в состояние ожидания
        b.states.Set(chatID, StateWaitingForTaskTitle)
        b.send(chatID, "Введи текст задачи следующим сообщением (или /cancel для отмены):")
        return
    }

    // Аргумент есть — добавляем сразу, как раньше
    task, err := b.repo.Add(chatID, arg)
    if err != nil {
        log.Printf("Add error: %v", err)
        b.send(chatID, "Ошибка при добавлении. Попробуй позже.")
        return
    }
    b.send(chatID, fmt.Sprintf("Задача #%d добавлена: %s", task.UserTaskID, task.Title))
}
```

Главное изменение: **раньше** при пустом arg возвращалась подсказка `Использование: /add <текст>`. **Теперь** бот переходит в состояние ожидания.

Это **обратно совместимое** изменение — старый сценарий (с аргументом) работает как раньше. Новый сценарий (без аргумента) даёт более удобный диалог.

## Ядро 6: что происходит между сообщениями

Сценарий по шагам:

1. **Пользователь пишет `/add`**
   - `handleMessage` → `handleCommand` (т.к. state = Idle) → `handleAdd("")` → `Set(chatID, WaitingForTaskTitle)` → ответ «Введи текст»
2. **Пользователь пишет `Купить молоко`**
   - `handleMessage` проверяет state → видит `WaitingForTaskTitle` → `handleStateful`
   - `handleStateful` видит state = `WaitingForTaskTitle` → вставляет задачу
   - `Set(chatID, StateIdle)` → ответ «Задача #1 добавлена»
3. **Пользователь пишет `/list`**
   - `handleMessage` проверяет state → `Idle` → `handleCommand` → `handleList`

Между сообщениями состояние **сохраняется в map**, и при следующем сообщении бот «помнит», что он в особом режиме для этого пользователя.

## Ядро 7: важные edge cases

### Что если пользователь после `/add` пишет другую команду?

```
Ты:    /add
Бот:   Введи текст задачи:
Ты:    /list
```

Сейчас у нас `handleStateful` обработает `/list` **как title задачи**, и создаст задачу с текстом «/list». Это **неудобно**.

Лучшее поведение — распознавать команды даже в состоянии:

```go
func (b *Bot) handleStateful(chatID int64, state State, text string) {
    text = strings.TrimSpace(text)

    // Команды всегда работают
    if strings.HasPrefix(text, "/") {
        b.states.Set(chatID, StateIdle)
        b.send(chatID, "Состояние сброшено, обрабатываю команду.")
        b.handleCommand(chatID, text)
        return
    }

    switch state {
    case StateWaitingForTaskTitle:
        // ...
    }
}
```

Теперь любая команда **выходит** из состояния и обрабатывается как обычная. Это «дружественное» поведение.

### Что если пользователь молчит после `/add`?

Если пользователь написал `/add`, передумал, и не пишет ничего — состояние `WaitingForTaskTitle` **остаётся навсегда**, пока пользователь не напишет что-нибудь.

Это **не критично** для нашего проекта. В production-ботах используют **TTL на состояния**: если состояние держится больше N минут — автосброс. Это можно сделать через отдельную горутину с тикером или через хранение времени установки в `StateStore`.

Для модуля 5 — без TTL.

### Что если бот перезапустился между `/add` и текстом задачи?

Состояние в памяти → потерялось. Пользователь напишет `Купить молоко`, бот его проигнорирует (или ответит «Не понимаю команду»). Это плохой UX.

Чтобы сохранять — надо хранить состояние **в БД или Redis**. Для нашего модуля 5 — оставляем в памяти, потому что:

- Перезапуски редкие (в продакшене)
- TTL естественный (после перезапуска состояние сбрасывается, что разумно)
- Для длинных форм — да, нужна персистентность; для двух шагов — нет

## Ядро 8: альтернатива — context.WithTimeout per state

Более продвинутый подход — хранить не просто `State`, а **состояние с дедлайном**:

```go
type StateEntry struct {
    State    State
    Deadline time.Time
}

type StateStore struct {
    mu     sync.Mutex
    states map[int64]StateEntry
}

func (s *StateStore) Get(chatID int64) State {
    s.mu.Lock()
    defer s.mu.Unlock()
    entry, ok := s.states[chatID]
    if !ok {
        return StateIdle
    }
    if time.Now().After(entry.Deadline) {
        delete(s.states, chatID)
        return StateIdle
    }
    return entry.State
}
```

И при `Set`:

```go
func (s *StateStore) Set(chatID int64, state State) {
    // ...
    s.states[chatID] = StateEntry{
        State:    state,
        Deadline: time.Now().Add(5 * time.Minute),
    }
}
```

Это даёт автоматический таймаут: если пользователь молчит 5 минут — состояние сбрасывается само. Полезно, но усложняет код.

**Для модуля 5 — оставь без TTL.** Для финального production-кода в Проекте 7 ты вернёшься к этой идее.

## Полный пример handleMessage

```go
func (b *Bot) handleMessage(msg *Message) {
    chatID := msg.Chat.ID
    text := msg.Text

    // Маршрутизация: состояние или команда?
    state := b.states.Get(chatID)
    if state != StateIdle {
        b.handleStateful(chatID, state, text)
        return
    }

    b.handleCommand(chatID, text)
}

func (b *Bot) handleCommand(chatID int64, text string) {
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
    case "/cancel":
        // в Idle — нечего отменять, но всё равно ответим
        b.send(chatID, "Нечего отменять.")
    default:
        b.handleUnknown(chatID)
    }
}
```

## Вопросы на понимание

1. Что такое **finite state machine**?
2. Где хранится состояние диалога — в памяти, в БД, или в чате?
3. Почему `State == 0` — это `StateIdle`, и почему это удобно?
4. Что произойдёт, если пользователь после `/add` пришлёт **другую команду**?
5. Зачем нужна команда `/cancel`?
6. Что произойдёт, если бот перезапустится во время диалога?
7. Какие плюсы и минусы у хранения состояния **в памяти** vs **в БД**?
8. Что такое **TTL** для состояния и зачем он нужен в production?
