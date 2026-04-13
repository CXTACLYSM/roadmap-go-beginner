# Модуль 2. GET — чтение задач — теория

## Контекст

В модуле 1 ты сделал сервер, который умеет отвечать на запросы текстовыми строками. Это разминка. В этом модуле начинается **настоящий API**: ты подключишь `TaskList` из Проекта 2/3 и научишься отдавать данные в формате JSON.

Будут два эндпоинта чтения:

- **`GET /tasks`** — список всех задач
- **`GET /tasks/{id}`** — одна задача по идентификатору

Это — самые простые эндпоинты в любом REST API. С них всегда начинается изучение нового сервера: «дай мне посмотреть, что там есть».

## Ядро 1: JSON в ответе

В Проекте 3 ты сериализовал задачи в JSON и записывал в файл. Теперь ты будешь сериализовать в JSON и записывать в `ResponseWriter`. Это **ровно то же самое**, только получатель другой.

```go
func listTasksHandler(w http.ResponseWriter, r *http.Request) {
    tasks := taskList.All()

    w.Header().Set("Content-Type", "application/json")
    err := json.NewEncoder(w).Encode(tasks)
    if err != nil {
        http.Error(w, "Ошибка сериализации", http.StatusInternalServerError)
        return
    }
}
```

Несколько новых деталей:

1. **`w.Header().Set("Content-Type", "application/json")`** — устанавливаем заголовок ответа. Этот заголовок говорит клиенту: «то, что я тебе сейчас пришлю в теле, — это JSON». Браузеры и клиентские библиотеки используют этот заголовок, чтобы автоматически парсить ответ.

2. **`json.NewEncoder(w).Encode(tasks)`** — это удобная альтернатива `json.Marshal` + `w.Write`. `NewEncoder` создаёт **энкодер**, привязанный к `w`, а `Encode` сериализует значение прямо туда. Внутри происходит то же самое, что у `Marshal` + `Write`, но без создания промежуточного `[]byte`.

3. **`http.Error(w, message, code)`** — стандартный способ вернуть ошибку клиенту. Делает три вещи: устанавливает `Content-Type: text/plain`, вызывает `WriteHeader(code)`, пишет сообщение в тело. После этого `return` обязателен, потому что мы не хотим продолжать запись в `w` дальше.

## Encoder vs Marshal — когда что использовать

**`json.Marshal(value)`** — возвращает `[]byte`. Удобно, если результат нужно куда-то ещё передать кроме записи (например, и записать, и захэшировать).

**`json.NewEncoder(w).Encode(value)`** — пишет напрямую в `Writer`. Удобно для HTTP-ответов и записи в файлы. Чуть эффективнее, потому что не создаёт промежуточный буфер.

Для HTTP-обработчиков **используй `Encoder`** — это идиома Go. Для «сделать JSON где-то в логике, а потом разобраться, что с ним делать» — `Marshal`.

## Ядро 2: Path patterns в Go 1.22+

Раньше для эндпоинтов вида `/tasks/{id}` (где `{id}` — это число) приходилось либо парсить путь руками, либо подключать внешний роутер (`chi`, `gorilla/mux`). С Go 1.22 это умеет **стандартный мультиплексор** — `http.ServeMux`.

Синтаксис регистрации:

```go
http.HandleFunc("GET /tasks", listTasksHandler)
http.HandleFunc("GET /tasks/{id}", getTaskHandler)
```

Несколько важных моментов:

1. **Метод указывается прямо в pattern-е**. Если придёт `POST /tasks`, и у тебя зарегистрирован только `GET /tasks` — клиент получит **405 Method Not Allowed**. Это огромное улучшение по сравнению со старым подходом, где приходилось проверять метод вручную.

2. **`{id}` — это path parameter**. Любое значение в этой позиции пути попадёт под этот обработчик, и его можно будет извлечь.

3. **Извлечение параметра**:

```go
func getTaskHandler(w http.ResponseWriter, r *http.Request) {
    idStr := r.PathValue("id")
    // idStr — это строка, нужно конвертировать в int
}
```

`r.PathValue("id")` возвращает значение path-параметра как **строку**. Если параметра нет — возвращает пустую строку.

## Парсинг ID и обработка ошибок

```go
func getTaskHandler(w http.ResponseWriter, r *http.Request) {
    idStr := r.PathValue("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "ID должен быть числом", http.StatusBadRequest)
        return
    }

    task, err := taskList.GetByID(id)
    if err != nil {
        http.Error(w, "Задача не найдена", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(task)
}
```

Здесь **два разных типа ошибок**, и они должны возвращать **разные статусы**:

- **`/tasks/abc`** — клиент прислал нечисловой ID. Это **400 Bad Request** — «ты прислал плохой запрос».
- **`/tasks/999`** (где задачи 999 не существует) — это **404 Not Found** — «такого ресурса нет».

Это типичный момент в дизайне API. Один и тот же эндпоинт может вернуть разные ошибки в зависимости от того, **что именно** не так с запросом. Хороший API возвращает **точный** статус, а не «всё, что не получилось — это 500».

## Ядро 3: что добавить в TaskList

Сейчас твой `TaskList` из Проекта 2/3 имеет методы `Add`, `MarkDone`, `Remove`, `findIndex`, `All`. Для модуля 2 тебе понадобится один новый метод: **`GetByID(id int) (Task, error)`**.

```go
func (tl *TaskList) GetByID(id int) (Task, error) {
    i := tl.findIndex(id)
    if i == -1 {
        return Task{}, fmt.Errorf("задача не найдена: %d", id)
    }
    return tl.Tasks[i], nil
}
```

Возвращаем **значение** `Task`, не указатель. Это идиоматично для Go: если структура небольшая, передавать копию проще и безопаснее, чем возиться с указателями.

При ошибке возвращаем **нулевую структуру** `Task{}` и осмысленную ошибку. Вызывающая сторона должна проверить ошибку и не использовать результат.

## Архитектура: обработчик, состояние, инициализация

В модуле 1 у тебя были **функции-обработчики** — простые функции, которые ничего не помнят между вызовами. Для эндпоинтов с данными это уже не работает: обработчику нужен доступ к `TaskList`, который существует **где-то** в программе.

Самый простой подход — **глобальная переменная**:

```go
var taskList = &TaskList{NextID: 1}

func listTasksHandler(w http.ResponseWriter, r *http.Request) {
    tasks := taskList.All()
    // ...
}
```

Это работает для маленьких проектов, но имеет минусы:

- Сложно тестировать — все обработчики жёстко привязаны к одной переменной
- Сложно иметь несколько независимых экземпляров (например, для двух разных API в одной программе)
- Глобальное состояние — вообще считается «code smell» в большинстве архитектур

**Лучший способ — методы на структуре-сервере:**

```go
type Server struct {
    tasks *TaskList
}

func (s *Server) listTasks(w http.ResponseWriter, r *http.Request) {
    tasks := s.tasks.All()
    // ...
}

func (s *Server) getTask(w http.ResponseWriter, r *http.Request) {
    // ...
}

func main() {
    server := &Server{tasks: &TaskList{NextID: 1}}

    http.HandleFunc("GET /tasks", server.listTasks)
    http.HandleFunc("GET /tasks/{id}", server.getTask)

    http.ListenAndServe(":8080", nil)
}
```

Заметь: `server.listTasks` — это **method value**. Когда ты передаёшь его в `HandleFunc`, Go автоматически создаёт функцию, которая привязана к этому `server`-у. Внутри `listTasks` ты получаешь доступ к `s.tasks`.

Этот подход сильно лучше глобальной переменной — для будущего проще тестировать, проще иметь разные экземпляры. **Используй его в этом модуле**, даже если кажется сложнее.

## Полный пример

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
)

type Task struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
    Done  bool   `json:"done"`
}

type TaskList struct {
    Tasks  []Task `json:"tasks"`
    NextID int    `json:"nextID"`
}

func (tl *TaskList) All() []Task {
    return tl.Tasks
}

func (tl *TaskList) GetByID(id int) (Task, error) {
    for _, t := range tl.Tasks {
        if t.ID == id {
            return t, nil
        }
    }
    return Task{}, fmt.Errorf("задача не найдена: %d", id)
}

type Server struct {
    tasks *TaskList
}

func (s *Server) listTasks(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    if err := json.NewEncoder(w).Encode(s.tasks.All()); err != nil {
        http.Error(w, "ошибка сериализации", http.StatusInternalServerError)
        return
    }
}

func (s *Server) getTask(w http.ResponseWriter, r *http.Request) {
    idStr := r.PathValue("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "ID должен быть числом", http.StatusBadRequest)
        return
    }

    task, err := s.tasks.GetByID(id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    if err := json.NewEncoder(w).Encode(task); err != nil {
        http.Error(w, "ошибка сериализации", http.StatusInternalServerError)
        return
    }
}

func main() {
    // Стартовые данные (в модуле 3 будем создавать через POST)
    server := &Server{
        tasks: &TaskList{
            Tasks: []Task{
                {ID: 1, Title: "Купить молоко", Done: false},
                {ID: 2, Title: "Позвонить маме", Done: true},
            },
            NextID: 3,
        },
    }

    http.HandleFunc("GET /tasks", server.listTasks)
    http.HandleFunc("GET /tasks/{id}", server.getTask)

    fmt.Println("Сервер запущен на http://localhost:8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        fmt.Println("Ошибка сервера:", err)
    }
}
```

Тест:

```bash
$ curl http://localhost:8080/tasks
[{"id":1,"title":"Купить молоко","done":false},{"id":2,"title":"Позвонить маме","done":true}]

$ curl http://localhost:8080/tasks/1
{"id":1,"title":"Купить молоко","done":false}

$ curl -i http://localhost:8080/tasks/999
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
...
задача не найдена: 999

$ curl -i http://localhost:8080/tasks/abc
HTTP/1.1 400 Bad Request
...
ID должен быть числом
```

`-i` показывает заголовки ответа вместе с телом — полезно для отладки.

## Ядро 4: ранний return при ошибках

Заметь паттерн в каждом обработчике:

```go
if err != nil {
    http.Error(w, "...", http.StatusXxx)
    return  // ← ОБЯЗАТЕЛЬНЫЙ
}
```

`return` после `http.Error` — **не опциональный**. Без него код пойдёт дальше и попытается **снова** записать в `w` — что приведёт к двойному ответу и предупреждению в логах:

```
http: superfluous response.WriteHeader call
```

Это типичная ошибка новичка в HTTP-обработчиках. **Запомни как мантру**: после любой записи ошибки — `return`.

## Вопросы на понимание

1. Зачем устанавливать заголовок `Content-Type: application/json`?
2. Чем `json.NewEncoder(w).Encode(...)` отличается от `json.Marshal(...) + w.Write(...)`?
3. Что такое **path parameter**? Как его извлечь в обработчике?
4. Почему `/tasks/abc` возвращает **400**, а `/tasks/999` (несуществующий) — **404**? В чём разница?
5. Зачем `return` после `http.Error`? Что произойдёт, если его забыть?
6. Какие плюсы у подхода с `Server`-структурой по сравнению с глобальной переменной?
7. Что выведет path pattern `GET /tasks/{id}`, если придёт запрос `POST /tasks/1`?
