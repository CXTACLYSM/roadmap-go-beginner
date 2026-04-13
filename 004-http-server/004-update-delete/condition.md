# Модуль 4. PUT и DELETE — условие

## Задание

Завершить CRUD REST API: добавить **PUT /tasks/{id}** для обновления и **DELETE /tasks/{id}** для удаления. Параллельно вынести парсинг ID в общий helper, чтобы не дублировать код.

## Что должна делать программа

```bash
# Обновление
$ curl -i -X PUT http://localhost:8080/tasks/1 \
    -H "Content-Type: application/json" \
    -d '{"title": "Обновлённое название", "done": true}'
HTTP/1.1 200 OK
Content-Type: application/json

{"id":1,"title":"Обновлённое название","done":true}

# Обновление несуществующей
$ curl -i -X PUT http://localhost:8080/tasks/999 \
    -H "Content-Type: application/json" \
    -d '{"title": "test", "done": false}'
HTTP/1.1 404 Not Found

задача не найдена: 999

# PUT с пустым title
$ curl -i -X PUT http://localhost:8080/tasks/1 \
    -H "Content-Type: application/json" \
    -d '{"title": "", "done": false}'
HTTP/1.1 400 Bad Request

# Удаление
$ curl -i -X DELETE http://localhost:8080/tasks/1
HTTP/1.1 204 No Content

(пустое тело)

# Повторное удаление того же
$ curl -i -X DELETE http://localhost:8080/tasks/1
HTTP/1.1 404 Not Found

задача не найдена: 1

# Удаление с нечисловым ID
$ curl -i -X DELETE http://localhost:8080/tasks/abc
HTTP/1.1 400 Bad Request

ID должен быть числом
```

## Требования

1. Реализован метод `(s *Server) updateTask(w, r)` для PUT
2. Реализован метод `(s *Server) deleteTask(w, r)` для DELETE
3. Добавлен метод `(tl *TaskList) Update(id int, title string, done bool) (Task, error)`:
   - Если задачи нет → возвращает ошибку
   - Если есть → обновляет `Title` и `Done`, возвращает обновлённую задачу
4. Зарегистрированы эндпоинты:
   - `PUT /tasks/{id}` → `updateTask`
   - `DELETE /tasks/{id}` → `deleteTask`
5. **Вынесен helper `parseID(w, r) (int, bool)`** — общий для всех трёх обработчиков с ID
6. `getTask`, `updateTask`, `deleteTask` используют этот helper, а не дублируют код
7. PUT использует **DTO** с двумя полями (Title, Done), не сам `Task`
8. PUT валидирует пустой title (как POST)
9. Успешный PUT → **200 OK** + JSON обновлённой задачи
10. PUT с несуществующим ID → **404 Not Found**
11. Успешный DELETE → **204 No Content**, **без тела**
12. DELETE с несуществующим ID → **404 Not Found**
13. Все эндпоинты из модулей 2-3 продолжают работать

## Шаги

1. Открой свой `app/`
2. **Сначала рефакторинг:** вынеси `parseID` в `server.go` или отдельный `helpers.go`
3. Применени `parseID` в существующем `getTask`. Запусти, проверь, что всё работает.
4. Добавь метод `Update` на `TaskList` в `tasklist.go`
5. Реализуй метод `updateTask` на `Server` (используй уже вынесенный `parseID`)
6. Зарегистрируй `PUT /tasks/{id}` в `main.go`
7. Запусти, прогоняй все сценарии PUT
8. Реализуй `deleteTask` (используй уже существующий `Remove` и `parseID`)
9. Зарегистрируй `DELETE /tasks/{id}`
10. Прогоняй все сценарии DELETE
11. Финальный прогон всех 5 эндпоинтов

## Эксперименты

1. **Идемпотентность PUT:** обнови задачу одним и тем же PUT три раза подряд. Состояние сервера после третьего вызова такое же, как после первого. Это — идемпотентность.

2. **Идемпотентность DELETE:** удали задачу, потом удали её ещё раз. Первый раз — 204, второй раз — 404. Состояние одинаковое — задачи нет. Это всё ещё идемпотентность (хотя статусы разные).

3. **Сравни с POST:** создай задачу через POST три раза подряд. Получишь три разные задачи с разными ID. Это **не** идемпотентно.

4. **Попробуй PUT с лишними полями:**
   ```bash
   curl -X PUT /tasks/1 -d '{"title": "test", "done": true, "id": 999, "evil": "hack"}'
   ```
   Что произошло с `id: 999`? **Ничего** — DTO его игнорирует. Задача 1 обновляется, ID не меняется. Это правильно: ID в URL — единственный источник истины.

5. **Попробуй PUT без поля done:**
   ```bash
   curl -X PUT /tasks/1 -d '{"title": "test"}'
   ```
   Что произошло с `done`? Он стал `false` (нулевое значение для bool). Это **особенность нашего PUT** — он заменяет **все** поля. Если хотел сохранить done, надо было передать оба поля. Это и есть разница между PUT и PATCH.

6. **Попробуй DELETE с телом:**
   ```bash
   curl -X DELETE /tasks/1 -H "Content-Type: application/json" -d '{"reason": "test"}'
   ```
   Тело **игнорируется** — наш сервер не читает его в DELETE-обработчике. Это нормально: семантически DELETE не должен иметь тела (хотя стандарт его не запрещает).

7. **После `WriteHeader(204)` попробуй записать что-то в `w`:**
   ```go
   w.WriteHeader(http.StatusNoContent)
   fmt.Fprintln(w, "пока!")  // эта строка проблема
   ```
   Что произошло? В логах сервера должно быть предупреждение. По стандарту HTTP, ответы со 204 не имеют тела.

8. **Попробуй POST на `/tasks/{id}` (то есть с ID в пути):**
   ```bash
   curl -X POST /tasks/1
   ```
   Что произошло? **405 Method Not Allowed**, потому что у тебя нет обработчика `POST /tasks/{id}`. У тебя есть `POST /tasks` (без ID), но это другой путь. Go различает их.

## Что показать Claude

После всего рефакторинга — покажи Claude свой `server.go` целиком и спроси: «есть ли ещё дублирование, которое стоит вынести в helper'ы?». Возможные кандидаты:

- **Установка `Content-Type: application/json` + `Encode`** — повторяется в `getTask`, `listTasks`, `createTask`, `updateTask`. Можно вынести в `writeJSON(w, status, value)`.
- **Парсинг тела с DTO** — повторяется в `createTask` и `updateTask`. Можно сделать `decodeJSON(r, &input)`.
- **Обработка ошибок «не найдено»** — повторяется в `getTask`, `updateTask`, `deleteTask`. Можно вынести в `respondNotFound(w, err)`.

Какие из этих рефакторингов сделать — твой выбор. **Один-два — хорошо**, но не превращай это в «архитектурный шедевр» — сейчас у тебя простой проект, и над-инжиниринг тут вреден. В Проекте 7 ты вернёшься к этим вопросам уже с пониманием, какая декомпозиция оправдана.
