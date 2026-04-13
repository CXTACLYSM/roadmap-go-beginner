# Модуль 4. CRUD из Go — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какой метод `*sql.DB` возвращает много строк?
2. Какой метод возвращает одну строку?
3. Какой метод выполняет запрос без результата (UPDATE/DELETE)?
4. Что значит `$1`, `$2` в SQL-запросах?
5. Какая ошибка означает «строка не найдена» в `database/sql`?

## Уровень 2: про шаблон `db.Query` + цикл

```go
rows, err := r.db.Query("SELECT id, title, done FROM tasks ORDER BY id")
if err != nil {
    return nil, fmt.Errorf("query: %w", err)
}
defer rows.Close()

var tasks []Task
for rows.Next() {
    var t Task
    if err := rows.Scan(&t.ID, &t.Title, &t.Done); err != nil {
        return nil, fmt.Errorf("scan: %w", err)
    }
    tasks = append(tasks, t)
}

if err := rows.Err(); err != nil {
    return nil, fmt.Errorf("rows iteration: %w", err)
}

return tasks, nil
```

Это **шаблон**, который ты будешь видеть везде. Запомни как мантру:

1. `Query` → `rows, err`
2. **Сразу** `defer rows.Close()`
3. Цикл `for rows.Next()` с `Scan` внутри
4. **После цикла** проверка `rows.Err()`
5. Возврат

Любое отклонение — ошибка. Особенно **первый** и **четвёртый** пункты.

## Уровень 3: про placeholders

```go
// PostgreSQL — нумерованные
db.QueryRow("SELECT * FROM tasks WHERE id = $1 AND done = $2", 5, true)

// Аргументы передаются ПОСЛЕ строки запроса, в том же порядке
```

Запомни: **`$1` — это первый аргумент после строки SQL**, `$2` — второй, и так далее. Можно ссылаться на один аргумент несколько раз:

```go
db.QueryRow("SELECT * FROM tasks WHERE title = $1 OR title = $1 || '?'", "test")
```

— здесь `$1` используется дважды, передаётся один аргумент.

В отличие от MySQL (`?, ?, ?`), PostgreSQL-стиль более явный.

## Уровень 4: про Scan и порядок аргументов

`Scan` записывает значения в переменные **в том же порядке**, что они идут в SELECT:

```go
// SQL: SELECT id, title, done
//        ^   ^      ^
//        |   |      └── &t.Done
//        |   └── &t.Title
//        └── &t.ID
err := rows.Scan(&t.ID, &t.Title, &t.Done)
```

Если перепутать — компилятор не поймает (типы могут совпадать), а runtime выдаст странные данные. Это распространённая ошибка.

**Правило:** держи `Scan` строго в том же порядке, что колонки в SELECT.

## Уровень 5: про sql.ErrNoRows

```go
import (
    "database/sql"
    "errors"
)

err := db.QueryRow("SELECT ...").Scan(...)
if err != nil {
    if errors.Is(err, sql.ErrNoRows) {
        // строки нет — это не «ошибка», это «не нашли»
        return ...нечто понятное...
    }
    // другие ошибки — это техническая проблема
    return fmt.Errorf("...: %w", err)
}
```

`sql.ErrNoRows` — это **переменная** (не функция), специальное значение, которое возвращается, когда `QueryRow.Scan` не нашёл строки.

Сравнивать только через `errors.Is`, не через `==`. Это потому, что в реальных проектах ошибка может быть **обёрнутой** через `%w` где-то по пути, и `==` тогда не сработает.

## Уровень 6: про INSERT ... RETURNING

```go
err := db.QueryRow(
    "INSERT INTO tasks (title) VALUES ($1) RETURNING id, title, done",
    title,
).Scan(&t.ID, &t.Title, &t.Done)
```

Несколько моментов:

- Для INSERT с RETURNING используется **`QueryRow`**, не `Exec` (потому что мы получаем строку обратно)
- В `Scan` передаём указатели на поля задачи — туда запишутся значения возвращённой строки
- `id` сгенерируется PostgreSQL (`BIGSERIAL`), `done` будет `false` (DEFAULT)
- В `RETURNING` явно перечисляем колонки (не `RETURNING *`)

## Уровень 7: про UPDATE ... RETURNING

```go
err := db.QueryRow(
    "UPDATE tasks SET title = $1, done = $2 WHERE id = $3 RETURNING id, title, done",
    title, done, id,
).Scan(&t.ID, &t.Title, &t.Done)

if errors.Is(err, sql.ErrNoRows) {
    // не нашли — задача с таким id не существует
}
```

Заметь:
- Все три параметра передаются позиционно — `title` для `$1`, `done` для `$2`, `id` для `$3`
- `RETURNING` после `WHERE`
- Если задачи нет — `Scan` вернёт `sql.ErrNoRows`, потому что UPDATE не нашёл строку для обновления

## Уровень 8: про db.Exec и RowsAffected

```go
result, err := r.db.Exec("DELETE FROM tasks WHERE id = $1", id)
if err != nil {
    return fmt.Errorf("delete: %w", err)
}

affected, err := result.RowsAffected()
if err != nil {
    return fmt.Errorf("rows affected: %w", err)
}

if affected == 0 {
    return fmt.Errorf("задача не найдена: %d", id)
}
```

`Exec` возвращает `sql.Result` — у него два метода:

- **`RowsAffected() (int64, error)`** — сколько строк затронуто
- **`LastInsertId() (int64, error)`** — id последней вставленной строки (работает для MySQL/SQLite, **не работает** для PostgreSQL — там используй `RETURNING`)

Для DELETE проверка `affected == 0` означает «ничего не нашлось по WHERE». Это твой способ узнать «не найдено».

## Уровень 9: про nil-слайс vs пустой слайс

```go
var tasks []Task
for rows.Next() {
    // ...
    tasks = append(tasks, t)
}
return tasks, nil
```

Если задач нет — `tasks` останется `nil`. Это **валидный** слайс в Go: `len(nil) == 0`, `range nil` — пустая итерация.

Когда вызывающий код использует это для JSON — `nil` сериализуется как `null`, а пустой слайс `[]Task{}` — как `[]`. Это разница, которую видит клиент.

Чтобы вернуть `[]` вместо `null`:

```go
tasks := []Task{}  // явный пустой слайс
for rows.Next() {
    // ...
    tasks = append(tasks, t)
}
return tasks, nil
```

Для нашего модуля 4 — без разницы. Для модуля 6 (HTTP) — может быть важно. Я рекомендую **сразу инициализировать пустым слайсом**, чтобы избежать сюрпризов.

## Уровень 10: про DELETE FROM tasks для воспроизводимости

В начале `main` для модуля 4:

```go
if _, err := db.Exec("DELETE FROM tasks"); err != nil {
    log.Fatalf("очистка таблицы: %v", err)
}
```

Это удалит все задачи перед каждым запуском, чтобы тест был воспроизводимым. **В реальном коде так делать не надо** — это игрушечный приём для модуля.

После DELETE можно сбросить sequence, чтобы id начинались с 1:

```go
if _, err := db.Exec("ALTER SEQUENCE tasks_id_seq RESTART WITH 1"); err != nil {
    log.Fatalf("restart sequence: %v", err)
}
```

— это сделает следующий INSERT с id=1, а не продолжит счётчик. Полезно для тестов.

## Если совсем тупик

Скажи Claude конкретно: «модуль 4 проекта 5, [конкретная проблема]». Самые частые ситуации:

- **«sql: expected N destination arguments in Scan»** → количество указателей в `Scan` не совпадает с колонками в SELECT
- **«sql: Scan error on column index ...»** → тип Go-переменной не совпадает с типом колонки
- **«too many connections»** → утечка соединений из-за забытого `defer rows.Close()`
- **«pq: column "..." does not exist»** → опечатка в имени колонки
- **«missing argument for $1»** → передал недостаточно аргументов в `Query`
