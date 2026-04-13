# Модуль 3. Save/Load и интеграция в REPL — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. На каком типе должны быть методы `Save` и `Load`? Почему?
2. Почему в этом модуле тебе пришлось сделать поля `TaskList` публичными?
3. В какой момент `main` нужно вызвать `Load` — до цикла или внутри?
4. После каких команд нужно сохранять? После каких — нет?
5. Что сделать, если `Load` упал с ошибкой?

## Уровень 2: про публичные поля

Сейчас у тебя:

```go
type TaskList struct {
    tasks  []Task
    nextID int
}
```

Поля приватные. `json.Marshal` их **не увидит** — он работает только с публичными (с большой буквы).

Меняй на:

```go
type TaskList struct {
    Tasks  []Task `json:"tasks"`
    NextID int    `json:"nextID"`
}
```

И **обнови все обращения внутри методов**: `tl.tasks` → `tl.Tasks`, `tl.nextID` → `tl.NextID`. Компилятор скажет, где что.

Тип `Task` тоже должен иметь теги:

```go
type Task struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
    Done  bool   `json:"done"`
}
```

Поля у `Task` уже были публичными, так что только теги.

## Уровень 3: про методы Save и Load

```go
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
```

Заметь: `json.MarshalIndent(tl, ...)` — передаём **сам указатель** `tl`. JSON-пакет знает, как сериализовать как структуры, так и указатели на структуры — выдаст одинаковый результат.

```go
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
```

Здесь `json.Unmarshal(data, tl)` — передаём **указатель** `tl`. `Unmarshal` обновит поля списка прямо на месте.

## Уровень 4: про инициализацию в main

```go
func main() {
    list := &TaskList{NextID: 1}

    if err := list.Load("tasks.json"); err != nil {
        fmt.Fprintln(os.Stderr, "Ошибка загрузки:", err)
        os.Exit(1)
    }
    // ... REPL ...
}
```

Несколько важных моментов:

- **`NextID: 1`** — стартовое значение. Если файла нет, `Load` ничего не поменяет, и `NextID` останется `1`. Если файл есть — `Load` перезапишет на загруженное значение.
- **`Load` до цикла** — потому что нужно прочитать данные **перед** тем, как пользователь начнёт работать.
- **При ошибке — выходим**. Не продолжаем работу, потому что иначе мы перезапишем потенциально валидный файл пустым списком.

## Уровень 5: про helper saveOrWarn

Чтобы не дублировать обработку ошибки в трёх местах:

```go
const dataFile = "tasks.json"

func saveOrWarn(list *TaskList) {
    if err := list.Save(dataFile); err != nil {
        fmt.Fprintln(os.Stderr, "Ошибка сохранения:", err)
    }
}
```

И использовать так:

```go
case "add":
    handleAdd(list, arg)
    saveOrWarn(list)
case "done":
    handleDone(list, arg)
    saveOrWarn(list)
case "remove":
    handleRemove(list, arg)
    saveOrWarn(list)
```

`list` (просмотр) и `exit` сохранение **не вызывают** — они ничего не меняют (хотя для exit можно вызвать на всякий случай, тогда не нужен `saveOrWarn` после изменяющих команд... но это сложнее логически. Проще оставить как описано).

## Уровень 6: про инициализацию NextID при загрузке

Тонкий момент. Если файла нет, `Load` возвращает `nil` сразу, **не трогая `tl`**. Это значит, что инициализация `&TaskList{NextID: 1}` сохраняется.

Если файл есть, `Unmarshal` **перезаписывает** `tl.NextID` загруженным значением. Тоже хорошо.

А что если файл есть, но в нём нет поля `nextID`? Тогда `Unmarshal` оставит `NextID` со значением **по умолчанию** для `int`, то есть `0`. И первая добавленная задача получит ID `0`, что не очень.

Это можно поправить так:

```go
func (tl *TaskList) Load(filename string) error {
    // ... как раньше ...
    if err := json.Unmarshal(data, tl); err != nil {
        return fmt.Errorf("парсинг: %w", err)
    }
    if tl.NextID == 0 {
        tl.NextID = 1  // на случай, если в файле не было nextID
    }
    return nil
}
```

Это **защитное программирование** — мы предполагаем, что файл может быть «не совсем правильным», и подстраховываемся. Хорошая привычка для работы с внешними данными.

## Уровень 7: про порядок операций

Правильный порядок в `main`:

1. Создать пустой `TaskList` с `NextID: 1`
2. Вызвать `Load` (он либо обновит данные из файла, либо оставит как есть)
3. Проверить ошибку `Load`
4. Создать сканер, начать REPL
5. После каждой изменяющей команды — `saveOrWarn`

Не наоборот. Если ты вызовешь `Load` после первого чтения команды — пользователь увидит пустой список и подумает, что данные пропали.

## Если совсем тупик

Скажи Claude конкретно: «модуль 3, не работает [конкретная штука]». Покажи код. Самые частые ситуации:

- «JSON получается пустой `{}`» → поля всё ещё приватные, забыл обновить
- «Программа падает при загрузке» → проверь обработку `os.ErrNotExist`
- «NextID становится 0 после загрузки** → проверь стартовое значение и логику в `Load`
- «Изменения теряются между запусками» → забыл вызвать `saveOrWarn` после команд
