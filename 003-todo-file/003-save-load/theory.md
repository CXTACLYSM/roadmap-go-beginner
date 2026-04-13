# Модуль 3. Save/Load и интеграция в REPL — теория

## Контекст

Ты прошёл два полигона — научился работать с JSON (модуль 1) и с файлами (модуль 2). Теперь нужно **соединить** эти навыки с твоим Проектом 2 и сделать приложение, которое реально сохраняет данные между запусками.

Это первая в твоей жизни задача **интеграции**: ты берёшь существующий код (TaskList из Проекта 2) и встраиваешь в него новую функциональность, **не сломав** старую. Это сильно отличается от написания с нуля и требует особой аккуратности.

## Ядро 1: методы Save и Load на TaskList

Логично сделать сохранение и загрузку **методами** типа `TaskList`. Они тесно связаны с состоянием списка — значит, должны жить рядом с ним.

```go
func (tl *TaskList) Save(filename string) error {
    data, err := json.MarshalIndent(tl, "", "  ")
    if err != nil {
        return fmt.Errorf("сериализация: %w", err)
    }
    return os.WriteFile(filename, data, 0644)
}

func (tl *TaskList) Load(filename string) error {
    data, err := os.ReadFile(filename)
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            return nil  // нет файла — нормально, оставляем пустым
        }
        return fmt.Errorf("чтение: %w", err)
    }
    return json.Unmarshal(data, tl)
}
```

Заметь несколько важных вещей:

1. **`Marshal` берёт сам `tl` (указатель на `TaskList`)** — это значит, что **вся структура** будет сериализована, включая внутренний слайс задач и счётчик `nextID`. Один вызов — весь объект.

2. **`Unmarshal` тоже работает с `tl`** — он прямо обновляет поля того списка, на котором вызвали метод. Это удобно: после `Load` ты сразу работаешь с обновлённым списком, не нужно ничего переприсваивать.

3. **`Load` для отсутствующего файла возвращает `nil`** — то же дизайнерское решение, что и в модуле 2. Это «нормальная ситуация».

4. **`Save` использует `MarshalIndent`** — потому что файл потенциально могут открыть и почитать руками. Для машинного формата хватило бы `Marshal`.

## Тонкость: JSON и приватные поля

Помнишь, в Проекте 2 ты сделал поля `TaskList` приватными (с маленькой буквы):

```go
type TaskList struct {
    tasks  []Task
    nextID int
}
```

Это была хорошая идея с точки зрения инкапсуляции — внешний код не должен лазить во внутреннее состояние. **Но** теперь возникает проблема: `json.Marshal` **не сериализует приватные поля**. Если оставить как есть, в файле будет:

```json
{}
```

— пустой объект. Никаких задач.

Что делать? Есть три варианта:

**Вариант A: Сделать поля публичными**

```go
type TaskList struct {
    Tasks  []Task `json:"tasks"`
    NextID int    `json:"nextID"`
}
```

Просто и работает. Минус — теряем инкапсуляцию: внешний код может писать в `list.Tasks` напрямую, в обход методов. Для нашего проекта это терпимо, но в идеальном мире — не очень.

**Вариант B: Свой `MarshalJSON` / `UnmarshalJSON`**

Можно научить `TaskList` сериализоваться особым способом, оставив поля приватными:

```go
type taskListDTO struct {
    Tasks  []Task `json:"tasks"`
    NextID int    `json:"nextID"`
}

func (tl *TaskList) MarshalJSON() ([]byte, error) {
    return json.Marshal(taskListDTO{Tasks: tl.tasks, NextID: tl.nextID})
}

func (tl *TaskList) UnmarshalJSON(data []byte) error {
    var dto taskListDTO
    if err := json.Unmarshal(data, &dto); err != nil {
        return err
    }
    tl.tasks = dto.Tasks
    tl.nextID = dto.NextID
    return nil
}
```

Это **правильный** способ для серьёзного кода. Но для учебного проекта — оверинжиниринг.

**Вариант C: Промежуточная структура внутри Save/Load**

```go
func (tl *TaskList) Save(filename string) error {
    dto := struct {
        Tasks  []Task `json:"tasks"`
        NextID int    `json:"nextID"`
    }{
        Tasks:  tl.tasks,
        NextID: tl.nextID,
    }
    data, err := json.MarshalIndent(dto, "", "  ")
    // ...
}
```

Тоже работает, тоже сохраняет инкапсуляцию.

**Для этого проекта используй Вариант A — публичные поля с JSON-тегами.** Это самый простой путь, и он соответствует тому, как ты будешь делать в Проекте 4 с DTO для веб-API. В Варианте B/C есть смысл, но для учебных целей лишний.

## Ядро 2: интеграция в REPL

После того как методы готовы, нужно встроить их в `main`. Две точки:

### 1. Загрузка при старте

```go
func main() {
    list := &TaskList{NextID: 1}

    if err := list.Load("tasks.json"); err != nil {
        fmt.Fprintln(os.Stderr, "Ошибка загрузки:", err)
        os.Exit(1)
    }

    // ... остальной REPL ...
}
```

Несколько важных моментов:

- **`Load` вызывается до цикла**, как часть инициализации
- **Если ошибка — выходим**, не продолжаем работу. Если файл повреждён, мы не хотим случайно перезаписать его пустым списком
- **Ошибку выводим в stderr**, как ты учил в Проекте 1
- **Если файла нет — `Load` вернёт `nil`**, и мы спокойно стартуем с пустым списком (это поведение мы заложили в `Load`)

### 2. Сохранение

Здесь нужно принять **дизайн-решение**: когда сохранять?

**Вариант 1: Только при exit**

```go
case "exit":
    if err := list.Save("tasks.json"); err != nil {
        fmt.Fprintln(os.Stderr, "Ошибка сохранения:", err)
    }
    fmt.Println("До встречи!")
    return
```

Плюсы: одно место сохранения, минимум I/O.
Минусы: если программа упадёт или будет убита (Ctrl+C, kill) — все изменения с момента последнего сохранения **потеряются**.

**Вариант 2: После каждой изменяющей команды**

```go
case "add":
    handleAdd(list, arg)
    if err := list.Save("tasks.json"); err != nil {
        fmt.Fprintln(os.Stderr, "Ошибка сохранения:", err)
    }
case "done":
    handleDone(list, arg)
    if err := list.Save("tasks.json"); err != nil { ... }
// и так далее для remove
```

Плюсы: данные **никогда** не теряются.
Минусы: больше I/O, дублирование кода (Save в трёх местах).

**Вариант 3: После каждой команды через универсальный дифф**

Можно вынести сохранение в отдельную функцию или даже сделать так, чтобы сами методы `TaskList.Add/MarkDone/Remove` сохраняли. Но это связывает слой данных с файловой системой — плохо для тестируемости.

**Для этого проекта используй Вариант 2.** Он самый надёжный: данные пользователя — это святое, и потерять их из-за упавшей программы нельзя. Дублирование `Save` в нескольких местах — приемлемая цена.

В Проекте 4 (HTTP) и Проекте 5 (БД) ты увидишь более продвинутые подходы. Сейчас держи просто.

## Ядро 3: рефакторинг с учётом сохранения

Дублирование `Save` в нескольких местах — повод подумать о **рефакторинге**. Можно сделать **обёртку** над handle-функциями:

```go
func runWithSave(list *TaskList, action func()) {
    action()
    if err := list.Save("tasks.json"); err != nil {
        fmt.Fprintln(os.Stderr, "Ошибка сохранения:", err)
    }
}

// в main:
case "add":
    runWithSave(list, func() { handleAdd(list, arg) })
case "done":
    runWithSave(list, func() { handleDone(list, arg) })
```

Это твоё **первое знакомство с функциями как значениями**. В Go функции — это полноценные значения: их можно передавать в аргументы, хранить в переменных, возвращать. `func() { ... }` — это **анонимная функция**, она создаётся прямо на месте вызова.

Это мощный приём, но для текущего модуля он **не обязателен**. Если хочешь — попробуй, это будет хорошее упражнение для подготовки к будущим проектам. Если нет — оставь дублирование Save, ничего страшного.

## Где хранить файл

Простой вариант: в **текущей рабочей директории** (то есть там, откуда ты запускаешь программу). Это значит, что если запустить `go run .` из разных папок, у тебя будут разные `tasks.json`.

Более продвинутые варианты:

- **Домашняя директория пользователя** — `~/.todo/tasks.json`. Через `os.UserHomeDir()`
- **Конфигурируемый путь через флаг командной строки** — `./todo --file=mytasks.json`
- **Через переменную окружения** — `TODO_FILE=mytasks.json ./todo`

Для текущего проекта — текущая папка. Просто и понятно. Усложнения — для следующих проектов.

## Полный пример (фрагмент)

```go
type TaskList struct {
    Tasks  []Task `json:"tasks"`
    NextID int    `json:"nextID"`
}

func (tl *TaskList) Save(filename string) error {
    data, err := json.MarshalIndent(tl, "", "  ")
    if err != nil {
        return fmt.Errorf("сериализация: %w", err)
    }
    if err := os.WriteFile(filename, data, 0644); err != nil {
        return fmt.Errorf("запись %s: %w", filename, err)
    }
    return nil
}

func (tl *TaskList) Load(filename string) error {
    data, err := os.ReadFile(filename)
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            return nil
        }
        return fmt.Errorf("чтение %s: %w", filename, err)
    }
    if err := json.Unmarshal(data, tl); err != nil {
        return fmt.Errorf("парсинг JSON: %w", err)
    }
    return nil
}

const dataFile = "tasks.json"

func main() {
    list := &TaskList{NextID: 1}

    if err := list.Load(dataFile); err != nil {
        fmt.Fprintln(os.Stderr, "Ошибка загрузки:", err)
        os.Exit(1)
    }

    scanner := bufio.NewScanner(os.Stdin)
    fmt.Println("To-Do приложение. Команды: add, list, done, remove, exit")

    for {
        fmt.Print("> ")
        if !scanner.Scan() {
            break
        }
        line := strings.TrimSpace(scanner.Text())
        if line == "" {
            continue
        }

        command, arg := parseCommand(line)

        switch command {
        case "add":
            handleAdd(list, arg)
            saveOrWarn(list)
        case "list":
            handleList(list)
        case "done":
            handleDone(list, arg)
            saveOrWarn(list)
        case "remove":
            handleRemove(list, arg)
            saveOrWarn(list)
        case "exit":
            fmt.Println("До встречи!")
            return
        default:
            fmt.Println("Неизвестная команда:", command)
        }
    }
}

func saveOrWarn(list *TaskList) {
    if err := list.Save(dataFile); err != nil {
        fmt.Fprintln(os.Stderr, "Ошибка сохранения:", err)
    }
}
```

Заметь: я вынес сохранение в маленький helper `saveOrWarn`, чтобы не дублировать обработку ошибки. Это элегантный компромисс — и не функции-значения (как в Варианте 3), и не три копипасты.

Команда `list` сохранение **не вызывает** — она ничего не меняет. Это маленькая, но важная оптимизация.

## Вопросы на понимание

1. Почему методы `Save` и `Load` живут на `TaskList`, а не как отдельные функции?
2. Что произойдёт, если поля `TaskList` оставить приватными и попытаться `Marshal`?
3. Какие три варианта решения этой проблемы ты знаешь? Какой выбран в этом модуле и почему?
4. В какой момент в `main` вызывается `Load`? Почему не позже?
5. Что произойдёт, если `Load` упадёт с ошибкой? Почему мы не игнорируем эту ошибку?
6. Какие плюсы и минусы у трёх стратегий сохранения (только при exit / после каждой команды / только при изменениях)? Какая выбрана и почему?
7. Почему `list` (просто показ) сохранение **не** вызывает?
