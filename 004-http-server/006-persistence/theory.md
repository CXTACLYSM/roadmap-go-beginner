# Модуль 6. Интеграция с файлом — теория

## Контекст

Поздравляю — ты дошёл до последнего модуля Проекта 4. У тебя есть рабочий REST API с защитой от race condition. Осталось одно: данные должны **переживать перезапуск сервера**.

Сейчас, когда ты останавливаешь сервер (Ctrl+C) и запускаешь снова, все задачи пропадают. Это потому, что они живут **только в памяти**. В Проекте 3 ты уже умеешь сохранять `TaskList` в JSON-файл — теперь нужно подключить эту способность к HTTP-серверу.

Этот модуль — **чистая интеграция**. Никаких новых концепций. Только аккуратное соединение того, что ты уже знаешь.

## Ядро 1: Save и Load на TaskList

Если ты ещё не перенёс `Save` и `Load` из Проекта 3 — сделай это сейчас. Методы те же, что были:

```go
func (tl *TaskList) Save(filename string) error {
    tl.mu.Lock()
    defer tl.mu.Unlock()

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
    tl.mu.Lock()
    defer tl.mu.Unlock()

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

**Главное отличие от Проекта 3** — здесь оба метода **захватывают мьютекс**. Это критично: пока мы сохраняем, никто не должен менять состояние, иначе мы запишем «полусломанное» состояние.

⚠️ **Тонкость с Marshal**: `json.Marshal(tl)` пытается сериализовать **все** публичные поля, включая мьютекс. Но `sync.Mutex` — это структура, которую нельзя осмысленно сериализовать. К счастью, наше поле `mu` — **с маленькой буквы**, и `Marshal` его игнорирует. Если бы оно было `Mu` — пришлось бы явно тегать `json:"-"`.

## Ядро 2: загрузка при старте

В `main`, **до** запуска сервера:

```go
const dataFile = "tasks.json"

func main() {
    tasks := &TaskList{NextID: 1}

    if err := tasks.Load(dataFile); err != nil {
        log.Fatalf("Ошибка загрузки данных: %v", err)
    }

    server := &Server{tasks: tasks}

    // ... регистрация обработчиков и ListenAndServe ...
}
```

Несколько важных моментов:

1. **`log.Fatalf`** — это новая функция. Делает две вещи: пишет в лог сообщение и **завершает программу с кодом 1**. Эквивалентно `log.Printf(...) + os.Exit(1)`.

2. **При ошибке загрузки — выходим**. Не пытаемся продолжить работу, потому что иначе мы потенциально перезапишем валидный файл пустыми данными. Это та же логика, что в Проекте 3.

3. **Если файла нет** — `Load` вернёт `nil` (это поведение мы заложили в Проекте 3 через `errors.Is(err, os.ErrNotExist)`). Сервер спокойно стартует с пустым списком.

## Ядро 3: где сохранять — в обработчике или в методе

Это **дизайнерский вопрос**, и у него есть два разумных ответа.

### Вариант A: сохранение в обработчике (рекомендуется)

```go
func (s *Server) createTask(w http.ResponseWriter, r *http.Request) {
    // ... парсинг, валидация ...
    task := s.tasks.Add(input.Title)

    if err := s.tasks.Save(dataFile); err != nil {
        log.Printf("Ошибка сохранения: %v", err)
    }

    // ... ответ клиенту ...
}
```

**Плюсы:**
- Слой данных (`TaskList`) ничего не знает про файл
- Можно в будущем использовать `TaskList` без сохранения (например, для тестов)
- Можно в одном обработчике сделать **несколько** изменений и сохранить **один раз** в конце

**Минусы:**
- Нужно не забыть `Save` в каждом изменяющем обработчике
- Дублирование

### Вариант B: сохранение внутри методов TaskList

```go
func (tl *TaskList) Add(title string) Task {
    tl.mu.Lock()
    defer tl.mu.Unlock()
    // ...
    if err := tl.saveLocked(); err != nil {
        log.Printf("Ошибка сохранения: %v", err)
    }
    return task
}
```

**Плюсы:**
- Невозможно забыть сохранение
- Меньше дублирования

**Минусы:**
- Слой данных **знает про файл** — это смешение слоёв
- Нельзя использовать `TaskList` без файла
- В тестах нужно либо мокать файл, либо тестировать с реальным файлом

**Для этого модуля используй вариант A** — он соответствует принципу разделения слоёв, который ты увидел в Проектах 2 и 3. В реальных приложениях оба подхода встречаются.

## Ядро 4: helper saveOrWarn

Чтобы не дублировать `if err := s.tasks.Save(...)` в каждом обработчике, выноси в helper, как ты делал в Проекте 3:

```go
func (s *Server) saveOrWarn() {
    if err := s.tasks.Save(dataFile); err != nil {
        log.Printf("Ошибка сохранения: %v", err)
    }
}
```

И вызывай:

```go
func (s *Server) createTask(w http.ResponseWriter, r *http.Request) {
    // ...
    task := s.tasks.Add(input.Title)
    s.saveOrWarn()
    // ...
}
```

**Что мы делаем при ошибке сохранения?** В этом примере — просто логируем и **продолжаем**. Это потому, что:

- Состояние в памяти **уже изменено**
- Возвращать клиенту ошибку поздно — изменение уже произошло
- Пользователь хочет видеть, что задача создалась

Это **компромисс**. В более серьёзном проекте можно было бы:

- Откатить изменение в памяти при ошибке записи
- Использовать паттерн «write-ahead log»
- Использовать настоящую базу данных, которая решает эту проблему через **транзакции**

Для нашего проекта — простое логирование. В Проекте 5 (БД) ты увидишь, как это делается «правильно».

## Ядро 5: где вызывать save — после каких операций

Save нужно вызывать **только** после изменяющих операций:

| Эндпоинт | Save? |
|----------|-------|
| GET /tasks | **Нет** (только чтение) |
| GET /tasks/{id} | **Нет** (только чтение) |
| POST /tasks | **Да** (создание) |
| PUT /tasks/{id} | **Да** (обновление) |
| DELETE /tasks/{id} | **Да** (удаление) |

Это очевидно, но напомню: **не вызывай Save после GET-эндпоинтов**. Это лишний дисковый I/O без причины.

## Ядро 6: тонкость с защитой Save

Раз `Save` сам захватывает мьютекс — вызывающий код **не должен** этого делать:

```go
// НЕПРАВИЛЬНО — двойная блокировка, deadlock
s.tasks.mu.Lock()
s.tasks.Save(dataFile)
s.tasks.mu.Unlock()

// ПРАВИЛЬНО — Save сам залочит и отпустит
s.tasks.Save(dataFile)
```

И ещё тонкость: **Save и обработчик не могут оба захватить мьютекс одновременно**, потому что Mutex нереентерабельный. Поэтому:

```go
// Это работает:
task := s.tasks.Add(input.Title)  // Add: Lock + ... + Unlock
s.saveOrWarn()                    // Save: Lock + ... + Unlock

// Это НЕ работает (deadlock):
s.tasks.mu.Lock()
defer s.tasks.mu.Unlock()
task := s.tasks.Add(input.Title)  // Add пытается Lock, который уже захвачен
```

В нашем дизайне обработчик **не лочит** мьютекс — он только вызывает методы `TaskList`, которые лочатся сами.

## Ядро 7: что с производительностью

Каждый изменяющий запрос теперь делает дополнительную операцию: записать **весь** файл на диск. Это:

- Медленно — особенно если задач много
- Создаёт нагрузку на диск
- Может проигрывать другому одновременному `Save` (если они идут параллельно)

**Это нормально для учебного проекта.** Для продакшена ты бы:

- Использовал базу данных, которая умеет писать только изменения
- Или делал «отложенное сохранение» — копил изменения в памяти и сбрасывал на диск раз в N секунд
- Или использовал append-only лог для быстрых записей

Все эти подходы — для следующих проектов. Сейчас задача — увидеть, как соединяются HTTP и persistence.

## Ядро 8: проблема параллельной записи в один файл

Если две `Save` идут одновременно (от двух разных запросов), они могут перезаписать друг друга. Mutex защищает от этого **внутри** одного процесса.

Но если ты запустишь **два экземпляра** сервера — они не будут знать друг о друге, и файл может быть повреждён. Эту проблему тоже решает база данных (через транзакции и блокировки на уровне строк).

В нашем случае — запускай один экземпляр.

## Полный пример обработчика после интеграции

```go
const dataFile = "tasks.json"

type Server struct {
    tasks *TaskList
}

func (s *Server) saveOrWarn() {
    if err := s.tasks.Save(dataFile); err != nil {
        log.Printf("Ошибка сохранения: %v", err)
    }
}

func (s *Server) createTask(w http.ResponseWriter, r *http.Request) {
    var input struct {
        Title string `json:"title"`
    }

    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        http.Error(w, "невалидный JSON", http.StatusBadRequest)
        return
    }
    if strings.TrimSpace(input.Title) == "" {
        http.Error(w, "title обязателен", http.StatusBadRequest)
        return
    }

    task := s.tasks.Add(input.Title)
    s.saveOrWarn()

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(task)
}

func (s *Server) updateTask(w http.ResponseWriter, r *http.Request) {
    id, ok := parseID(w, r)
    if !ok {
        return
    }

    var input struct {
        Title string `json:"title"`
        Done  bool   `json:"done"`
    }
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        http.Error(w, "невалидный JSON", http.StatusBadRequest)
        return
    }
    if strings.TrimSpace(input.Title) == "" {
        http.Error(w, "title обязателен", http.StatusBadRequest)
        return
    }

    task, err := s.tasks.Update(id, input.Title, input.Done)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    s.saveOrWarn()

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(task)
}

func (s *Server) deleteTask(w http.ResponseWriter, r *http.Request) {
    id, ok := parseID(w, r)
    if !ok {
        return
    }

    if err := s.tasks.Remove(id); err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    s.saveOrWarn()

    w.WriteHeader(http.StatusNoContent)
}

func main() {
    tasks := &TaskList{NextID: 1}

    if err := tasks.Load(dataFile); err != nil {
        log.Fatalf("Ошибка загрузки: %v", err)
    }

    server := &Server{tasks: tasks}

    http.HandleFunc("GET /tasks", server.listTasks)
    http.HandleFunc("GET /tasks/{id}", server.getTask)
    http.HandleFunc("POST /tasks", server.createTask)
    http.HandleFunc("PUT /tasks/{id}", server.updateTask)
    http.HandleFunc("DELETE /tasks/{id}", server.deleteTask)

    log.Println("Сервер запущен на http://localhost:8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatalf("Ошибка сервера: %v", err)
    }
}
```

Заметь, что я использую `log.Println` и `log.Fatalf` вместо `fmt.Println` и `os.Exit`. **Пакет `log`** удобнее для серверных приложений: добавляет временную метку, отправляет в `os.Stderr` по умолчанию, и `Fatalf` сама делает `Exit(1)` после печати.

Привыкай: в серверном коде используют `log`, не `fmt`.

## Вопросы на понимание

1. Где должен быть вызов `Load` — до запуска сервера или внутри обработчика? Почему?
2. Что делать, если `Load` упал с ошибкой?
3. После каких HTTP-методов нужно вызывать `Save`? После каких — нет?
4. Почему `Save` сам захватывает мьютекс, а обработчик — нет?
5. Что произойдёт, если обработчик попытается **тоже** захватить мьютекс перед вызовом `Add` или `Save`?
6. Какие два варианта дизайна (где звать `Save`)? Какой выбран и почему?
7. Что делает `log.Fatalf`?
