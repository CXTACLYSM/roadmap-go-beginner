# Модуль 4. PUT и DELETE — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какой статус возвращать при успешном обновлении? При успешном удалении?
2. Должно ли быть тело у ответа DELETE?
3. Как извлечь ID из URL `/tasks/5` в обработчике PUT?
4. Что общего между парсингом ID в `getTask`, `updateTask` и `deleteTask`?

## Уровень 2: про parseID helper

Дублирование в трёх обработчиках:

```go
idStr := r.PathValue("id")
id, err := strconv.Atoi(idStr)
if err != nil {
    http.Error(w, "ID должен быть числом", http.StatusBadRequest)
    return
}
```

Выноси в helper:

```go
func parseID(w http.ResponseWriter, r *http.Request) (int, bool) {
    idStr := r.PathValue("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "ID должен быть числом", http.StatusBadRequest)
        return 0, false
    }
    return id, true
}
```

Использование:

```go
func (s *Server) getTask(w http.ResponseWriter, r *http.Request) {
    id, ok := parseID(w, r)
    if !ok {
        return  // ошибка уже отправлена внутри parseID
    }
    // основная логика с id
}
```

Заметь дизайн: `parseID` сам пишет ошибку в `w` и сообщает «всё плохо» через `bool`. Вызывающий просто `return`-ит. Это типичный паттерн для HTTP-helper'ов в Go.

Альтернатива — возвращать `(int, error)` и обрабатывать в каждом месте, но это менее удобно.

## Уровень 3: про метод Update

Добавь в `tasklist.go`:

```go
func (tl *TaskList) Update(id int, title string, done bool) (Task, error) {
    i := tl.findIndex(id)
    if i == -1 {
        return Task{}, fmt.Errorf("задача не найдена: %d", id)
    }
    tl.Tasks[i].Title = title
    tl.Tasks[i].Done = done
    return tl.Tasks[i], nil
}
```

Помнишь из Проекта 2 — изменение **через индекс**, а не через копию из range. Если ты сейчас обновишь `tl.Tasks[i]`, это будет реальное изменение.

## Уровень 4: про DTO для PUT

Локальная структура для тела PUT:

```go
var input struct {
    Title string `json:"title"`
    Done  bool   `json:"done"`
}
```

Здесь два поля, потому что PUT — **полная замена**. Клиент должен явно указать оба.

ID в DTO **нет** — он берётся из URL через `parseID`.

## Уровень 5: про статус 204

```go
w.WriteHeader(http.StatusNoContent)
// никаких Encode, никаких Fprintln, никакой записи в w после этого
```

204 — это «всё ОК, тело отсутствует». Если ты после этого попытаешься что-то записать в `w` — Go ругнётся в логи.

## Уровень 6: про DELETE без тела

В `deleteTask` ты **не парсишь** тело запроса. Никакого `json.Decoder`, никакого DTO. Просто:

1. Парси ID из пути (через `parseID`)
2. Вызови `s.tasks.Remove(id)`
3. При ошибке — 404
4. При успехе — 204

Минимум кода, и это правильно.

## Уровень 7: про идемпотентность DELETE и разные статусы

Тебе может показаться странным: первый DELETE возвращает 204, второй — 404, и при этом я говорю «оба идемпотентны». В чём логика?

**Идемпотентность — это про состояние сервера, а не про ответ.**

После первого DELETE состояние: задачи 5 нет.
После второго DELETE состояние: задачи 5 нет.

Состояние одинаковое → идемпотентно.

Ответы могут отличаться (404 второй раз — это «нечего удалять»), но **сервер не накапливает ничего** при повторных вызовах. В этом и смысл.

Сравни с POST: каждый вызов **увеличивает** число задач. Это не идемпотентно.

## Уровень 8: про регистрацию обработчиков

В `main`:

```go
http.HandleFunc("GET /tasks", server.listTasks)
http.HandleFunc("GET /tasks/{id}", server.getTask)
http.HandleFunc("POST /tasks", server.createTask)
http.HandleFunc("PUT /tasks/{id}", server.updateTask)
http.HandleFunc("DELETE /tasks/{id}", server.deleteTask)
```

Пять обработчиков на два пути. Go различает их по методу.

Если кто-то пришлёт `POST /tasks/1` — у тебя нет такого pattern-а, и Go вернёт **405 Method Not Allowed**.

## Уровень 9: про опциональный рефакторинг

Если ты хочешь дополнительно вынести запись JSON-ответа в helper:

```go
func writeJSON(w http.ResponseWriter, status int, value any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(value); err != nil {
        log.Printf("ошибка кодирования JSON: %v", err)
    }
}
```

Использование:

```go
writeJSON(w, http.StatusOK, tasks)
writeJSON(w, http.StatusCreated, newTask)
```

Это убирает повторение в каждом обработчике. **Делай, если хочешь** — это упражнение в декомпозиции, не обязательно для зачёта модуля.

`any` в Go — это псевдоним для `interface{}` (который тут означает «любой тип»).

## Если совсем тупик

Скажи Claude конкретно: «модуль 4 проекта 4, [конкретная проблема]». Самые частые ситуации:

- «Update обновляет копию, не оригинал» → используй `tl.Tasks[i].Title = ...`, не `t.Title = ...` через range
- «DELETE возвращает 200 с пустым телом** → используй `WriteHeader(http.StatusNoContent)` явно, иначе Go запишет 200 при первом обращении к `w`
- «PUT не находит задачу с правильным ID» → проверь, что `findIndex` сравнивает по ID, а не по индексу
- «POST на `/tasks/1` возвращает 200» → проверь, что у тебя нет случайно зарегистрированного `"POST /tasks/{id}"`
