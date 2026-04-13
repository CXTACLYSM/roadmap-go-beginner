# Модуль 4. PUT и DELETE — теория

## Контекст

После модуля 3 у тебя есть R и C из CRUD: чтение и создание. В этом модуле добавятся U (Update) и D (Delete) — обновление и удаление. После него твой API будет **полным**.

Но это не просто «ещё два обработчика». В этом модуле появляются несколько важных концептов:

- **Разница между PUT и PATCH** — два подхода к обновлению
- **Идемпотентность** — почему PUT и DELETE безопасно повторять
- **Статус 204 No Content** — когда тело ответа не нужно
- **Дублирование кода** — три обработчика будут парсить ID одинаково. Это сигнал к рефакторингу.

## Ядро 1: PUT — полная замена ресурса

`PUT /tasks/{id}` — это «положи **этот** объект в это место». Если ресурс есть — он заменяется. Если ресурса нет — поведение зависит от соглашения (можно вернуть 404, можно создать).

Пример запроса:

```http
PUT /tasks/5 HTTP/1.1
Content-Type: application/json

{"title": "Обновлённый заголовок", "done": true}
```

Семантика: «у задачи 5 теперь такие свойства: title=..., done=...». Все остальные поля (если они есть в исходном объекте) считаются **сброшенными** в значения по умолчанию.

В нашем случае структура простая, и PUT почти эквивалентен `PATCH` (см. ниже). Но если бы у задачи было 10 полей, разница была бы заметна.

## Ядро 2: PUT vs PATCH

| | PUT | PATCH |
|---|-----|-------|
| Семантика | «Замени **весь** ресурс» | «Измени **только указанные поля**» |
| Тело запроса | Полный объект | Только меняемые поля |
| Идемпотентность | Да | Обычно нет |
| Когда использовать | Простые ресурсы, замена | Сложные ресурсы, частичные обновления |

**PATCH** более гибок, но сложнее в реализации: тебе нужно понимать, какие поля **присутствуют** в запросе (а не просто `false`/`""`), и обновлять только их. Это требует особой обработки JSON, потому что в Go нет встроенного способа отличить «поле не было в JSON» от «поле было `null`».

Для нашего проекта мы делаем **PUT**. Это проще, и для маленькой структуры `Task` функционально эквивалентно PATCH.

## Ядро 3: реализация PUT-обработчика

```go
func (s *Server) updateTask(w http.ResponseWriter, r *http.Request) {
    // 1. Парсим ID из пути
    idStr := r.PathValue("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "ID должен быть числом", http.StatusBadRequest)
        return
    }

    // 2. Парсим тело
    var input struct {
        Title string `json:"title"`
        Done  bool   `json:"done"`
    }
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        http.Error(w, "невалидный JSON", http.StatusBadRequest)
        return
    }

    // 3. Валидация
    if strings.TrimSpace(input.Title) == "" {
        http.Error(w, "title обязателен", http.StatusBadRequest)
        return
    }

    // 4. Обновление
    task, err := s.tasks.Update(id, input.Title, input.Done)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    // 5. Ответ
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(task)
}
```

Заметь несколько моментов:

1. **Тот же DTO-подход**, что в модуле 3, но теперь с двумя полями: `Title` и `Done`. ID не входит в DTO — он берётся из URL.

2. **`Update` — новый метод на `TaskList`**:
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
   Возвращает обновлённую задачу или ошибку, если задачи нет.

3. **Статус 200 OK** — для успешного обновления (не 201, потому что ничего не создавали).

## Ядро 4: DELETE — удаление

```go
func (s *Server) deleteTask(w http.ResponseWriter, r *http.Request) {
    idStr := r.PathValue("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "ID должен быть числом", http.StatusBadRequest)
        return
    }

    if err := s.tasks.Remove(id); err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    // Успех — пустой ответ со статусом 204
    w.WriteHeader(http.StatusNoContent)
}
```

Несколько моментов:

1. **Нет тела для парсинга** — DELETE обычно не имеет тела
2. **Метод `Remove` уже есть** на `TaskList` из Проекта 2 (модуль 4)
3. **Статус 204 No Content** — для успешного удаления. Тело ответа **отсутствует**.

## Ядро 5: статус 204 No Content

204 — это «всё прошло хорошо, но мне нечего тебе вернуть в теле». Используется для:

- DELETE — успешное удаление
- PUT/PATCH — успешное обновление, если клиенту не нужен обновлённый объект
- POST — успешное действие, которое не создаёт нового ресурса (например, отправка email)

**Важно: при 204 НЕ должно быть тела.** Если ты после `WriteHeader(204)` напишешь что-то в `w` — это нарушение HTTP-стандарта, и Go может выдать предупреждение или вообще отбросить твои данные.

Поэтому шаблон для 204:

```go
w.WriteHeader(http.StatusNoContent)
// никаких Encode, никаких Fprintln после этого
```

## Ядро 6: идемпотентность на практике

Помнишь из модуля 1, что **PUT и DELETE — идемпотентные**? Что это значит **на практике**?

Идемпотентный = повторный вызов даёт **тот же эффект на состояние сервера**, что и первый. Не обязательно тот же ответ, но тот же эффект.

**Пример с PUT:**

```bash
# Первый вызов
$ curl -X PUT /tasks/5 -d '{"title": "test", "done": true}'
{"id":5,"title":"test","done":true}    # 200 OK

# Второй вызов (тот же запрос)
$ curl -X PUT /tasks/5 -d '{"title": "test", "done": true}'
{"id":5,"title":"test","done":true}    # 200 OK
```

После первого вызова состояние: задача `{id:5, title:"test", done:true}`. После второго — то же самое. Сервер ничего не «накопил». **Это и есть идемпотентность.**

**Пример с DELETE:**

```bash
# Первый вызов
$ curl -X DELETE /tasks/5
# 204 No Content

# Второй вызов
$ curl -X DELETE /tasks/5
# 404 Not Found (задача уже удалена)
```

Заметь: ответы **разные** (204 в первый раз, 404 во второй). Но **состояние сервера одинаковое** — задачи 5 нет. Это **тоже идемпотентность**: второй вызов не делает «ещё больше удаления».

**Сравни с POST:**

```bash
# Первый вызов
$ curl -X POST /tasks -d '{"title": "test"}'
{"id":3,"title":"test","done":false}    # 201

# Второй вызов (тот же запрос)
$ curl -X POST /tasks -d '{"title": "test"}'
{"id":4,"title":"test","done":false}    # 201
```

После первого вызова — одна задача. После второго — **две**. Это **не идемпотентно**.

**Зачем тебе это знать?** Потому что:

1. Идемпотентные запросы **безопасно повторять**, если связь оборвалась. Клиент может смело попробовать ещё раз.
2. Прокси, кэши, балансировщики нагрузки **рассчитывают** на эту семантику.
3. Если ты неправильно реализуешь свой PUT (например, делаешь его неидемпотентным), ты ломаешь экосистему.

## Ядро 7: дублирование кода

Посмотри на свои четыре обработчика: `getTask`, `updateTask`, `deleteTask`. В **трёх** из них есть **одинаковый код** парсинга ID:

```go
idStr := r.PathValue("id")
id, err := strconv.Atoi(idStr)
if err != nil {
    http.Error(w, "ID должен быть числом", http.StatusBadRequest)
    return
}
```

Это **дублирование**, и оно просится в helper:

```go
func parseID(r *http.Request, w http.ResponseWriter) (int, bool) {
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
    id, ok := parseID(r, w)
    if !ok {
        return  // ошибка уже отправлена
    }
    // дальше — основная логика
}
```

Заметь дизайн: функция возвращает **`(int, bool)`** и сама записывает ошибку в `w`, если что-то не так. Вызывающий код проверяет `ok` и просто `return`-ит.

Это типичный паттерн в Go-обработчиках. Альтернатива — возвращать `error` и обрабатывать его в каждом месте, но это менее удобно.

**Сделай этот рефакторинг в этом модуле.** Это часть задания.

## Полный пример обработчиков и регистрации

```go
// Helper для парсинга ID
func parseID(w http.ResponseWriter, r *http.Request) (int, bool) {
    idStr := r.PathValue("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "ID должен быть числом", http.StatusBadRequest)
        return 0, false
    }
    return id, true
}

// Обновление
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

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(task)
}

// Удаление
func (s *Server) deleteTask(w http.ResponseWriter, r *http.Request) {
    id, ok := parseID(w, r)
    if !ok {
        return
    }

    if err := s.tasks.Remove(id); err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    w.WriteHeader(http.StatusNoContent)
}

// Регистрация в main
http.HandleFunc("PUT /tasks/{id}", server.updateTask)
http.HandleFunc("DELETE /tasks/{id}", server.deleteTask)
```

## Финальная карта всех эндпоинтов

После модуля 4 у твоего API будет:

| Метод | Путь | Действие | Успех | Главная ошибка |
|-------|------|----------|-------|----------------|
| GET | /tasks | Список | 200 | — |
| GET | /tasks/{id} | Одна задача | 200 | 404 |
| POST | /tasks | Создать | 201 | 400 |
| PUT | /tasks/{id} | Обновить | 200 | 404, 400 |
| DELETE | /tasks/{id} | Удалить | 204 | 404 |

Это и есть классический **CRUD REST API**. Ты только что написал свой первый.

## Вопросы на понимание

1. В чём разница между PUT и PATCH?
2. Почему мы делаем PUT, а не PATCH? Когда выбор был бы другой?
3. Что такое **идемпотентность**? Чем PUT и DELETE отличаются от POST в этом плане?
4. Почему DELETE при повторном вызове возвращает разные статусы (204 первый раз, 404 второй), но всё ещё считается идемпотентным?
5. Что такое статус **204 No Content** и когда его использовать?
6. Почему **нельзя** ничего писать в `w` после `WriteHeader(204)`?
7. Зачем выносить парсинг ID в helper-функцию `parseID`? Какие плюсы?
