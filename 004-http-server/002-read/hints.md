# Модуль 2. GET — чтение задач — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какой пакет нужен для JSON?
2. Как сказать клиенту в HTTP-ответе, что в теле — JSON?
3. Как зарегистрировать обработчик, который реагирует **только на GET**?
4. Как извлечь значение `{id}` из пути запроса?
5. Какой статус возвращать для нечислового ID? Какой — для несуществующего?

## Уровень 2: про path patterns

Синтаксис в Go 1.22+:

```go
http.HandleFunc("GET /tasks", listTasks)
http.HandleFunc("GET /tasks/{id}", getTask)
```

- **Метод и путь разделены пробелом**, метод — заглавными буквами
- **`{id}`** — placeholder для path-параметра. Можно называть как угодно.
- Если придёт другой метод (например, POST на `/tasks`), и POST не зарегистрирован — Go сам вернёт **405 Method Not Allowed**

Извлечение параметра:

```go
idStr := r.PathValue("id")  // строка "42" или ""
```

Имя в `PathValue` должно совпадать с именем в фигурных скобках pattern-а.

## Уровень 3: про json.NewEncoder

Шаблон записи JSON в ответ:

```go
w.Header().Set("Content-Type", "application/json")
err := json.NewEncoder(w).Encode(value)
if err != nil {
    http.Error(w, "ошибка сериализации", http.StatusInternalServerError)
    return
}
```

`json.NewEncoder(w)` создаёт энкодер, привязанный к `w`. `Encode(value)` сериализует значение и записывает прямо в `w`.

Это удобнее, чем `Marshal + Write`, потому что:
- Не создаётся промежуточный `[]byte`
- Меньше кода
- Идиоматичнее

## Уровень 4: про http.Error

`http.Error` — это удобный helper для возврата ошибок:

```go
http.Error(w, "сообщение", http.StatusBadRequest)
```

Эта одна строка делает три вещи:
1. Устанавливает `Content-Type: text/plain; charset=utf-8`
2. Вызывает `WriteHeader(http.StatusBadRequest)`
3. Записывает сообщение в тело ответа с переводом строки

**После `http.Error` обязательно `return`!** Иначе код пойдёт дальше и попытается ещё раз записать в `w`.

## Уровень 5: про статусы

Используй **константы из пакета `http`**, а не голые числа:

```go
http.StatusOK                     // 200
http.StatusCreated                // 201
http.StatusNoContent              // 204
http.StatusBadRequest             // 400
http.StatusNotFound               // 404
http.StatusMethodNotAllowed       // 405
http.StatusInternalServerError    // 500
```

Это лучше, чем `400` или `404`, потому что:
- В коде сразу понятно, какой статус возвращается
- Опечатку поймает компилятор
- IDE подсказывает доступные варианты

## Уровень 6: про метод-обработчик на структуре

```go
type Server struct {
    tasks *TaskList
}

func (s *Server) listTasks(w http.ResponseWriter, r *http.Request) {
    // здесь s.tasks доступен
}
```

Регистрация:

```go
server := &Server{tasks: &TaskList{NextID: 1}}
http.HandleFunc("GET /tasks", server.listTasks)
```

`server.listTasks` — это **method value**. Go создаёт функцию, которая «помнит» конкретный `server`-экземпляр. Внутри обработчика `s` будет указывать на этот экземпляр.

## Уровень 7: про порядок установки заголовка и записи

Правильный порядок:

```go
w.Header().Set("Content-Type", "application/json")  // 1. заголовки
// (опционально: w.WriteHeader(http.StatusOK))      // 2. статус
json.NewEncoder(w).Encode(tasks)                    // 3. тело
```

Если установить заголовок **после** записи в тело — он будет проигнорирован, и Go выведет предупреждение в логи. Это потому, что заголовки уходят на провод **до** тела, и поменять их потом нельзя.

Для статуса 200 (по умолчанию) `WriteHeader` можно опустить — Go сам пошлёт 200 при первой записи в тело.

## Уровень 8: про -i в curl

Полезный флаг:

```bash
curl -i http://localhost:8080/tasks
```

`-i` (от include) — печатает **заголовки ответа вместе с телом**. Незаменимо для отладки.

Ещё больше деталей — `-v` (verbose), показывает заголовки **запроса и ответа** + дополнительная диагностика TLS/соединения.

## Уровень 9: про разделение на файлы

К концу модуля 2 у тебя должно быть несколько файлов в `app/`:

```
app/
├── go.mod
├── main.go         ← main, инициализация, регистрация обработчиков
├── server.go       ← тип Server и его методы-обработчики
├── task.go         ← тип Task
└── tasklist.go     ← тип TaskList и его методы
```

Все в `package main`. Разделение по логике, как в Проекте 3.

## Если совсем тупик

Скажи Claude конкретно: «модуль 2 проекта 4, [конкретная проблема]». Покажи код. Самые частые ситуации:

- «`r.PathValue` всегда пустая строка** → проверь, что pattern содержит `{id}` и имя совпадает
- «Browser показывает HTML вместо JSON** → забыл `Content-Type: application/json`
- «`http: superfluous response.WriteHeader call`» → забыл `return` после `http.Error`
- «POST возвращает 200, а должен 405» → ты не указал метод в pattern (написал `/tasks` вместо `GET /tasks`)
