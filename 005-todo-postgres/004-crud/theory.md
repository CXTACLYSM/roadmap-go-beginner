# Модуль 4. CRUD из Go — теория

## Контекст

В модуле 3 ты сделал «hello world»: подключился к PostgreSQL и выполнил `SELECT 1`. В модуле 4 — настоящая работа. Ты перепишешь **все** методы `TaskList` так, чтобы данные жили в PostgreSQL, а не в памяти.

После этого модуля у тебя будет **полноценный SQL-репозиторий**, готовый к интеграции с HTTP-сервером (модуль 6).

## Ядро 1: TaskRepository вместо TaskList

Сначала о терминологии. В Проектах 2-4 ты называл этот тип `TaskList` — список задач в памяти. Теперь, когда данные в БД, более точное имя — **репозиторий**. Это паттерн из Domain-Driven Design: репозиторий — это **абстракция над хранилищем**, которая знает, как сохранить и достать сущность из «где-то».

```go
type TaskRepository struct {
    db *sql.DB
}

func NewTaskRepository(db *sql.DB) *TaskRepository {
    return &TaskRepository{db: db}
}
```

Несколько важных моментов:

1. **Поле `db *sql.DB`** — указатель на пул соединений. Создаётся снаружи и передаётся в конструктор.
2. **`NewTaskRepository`** — конструктор. Не обязателен (можно создавать через литерал), но идиоматичен в Go.
3. **Никакого `Tasks []Task`, никакого `NextID`** — эти поля больше не нужны. Состояние живёт в БД.
4. **Никакого `sync.Mutex`** — БД сама управляет конкурентным доступом через транзакции и блокировки.

Это огромное упрощение. Просто посмотри на размер — два поля вместо четырёх, и одно из них — указатель на чужой объект.

## Ядро 2: метод GetAll через db.Query

`db.Query` — это запрос, возвращающий **много** строк. В отличие от `QueryRow` (одна строка), здесь нужно итерировать.

```go
func (r *TaskRepository) GetAll() ([]Task, error) {
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
}
```

Разберём по частям:

1. **`db.Query(sql)`** возвращает `*sql.Rows` и ошибку. Ошибка может быть из-за плохого SQL, разрыва соединения, неправильных типов и так далее.

2. **`defer rows.Close()`** — **обязательно**. `Rows` держит соединение из пула, и без `Close` соединение **не вернётся в пул**. Со временем пул закончится, и сервер встанет. Это критичная ошибка, и Go-сообщество часто говорит «всегда делай `defer rows.Close()` сразу после успешного `Query`».

3. **Цикл `for rows.Next()`** — итерация по строкам. `Next()` возвращает `false`, когда строки кончились или произошла ошибка. На каждой итерации:
   - Создаём пустую `Task`
   - Через `rows.Scan(...)` копируем колонки в поля
   - Добавляем в результирующий слайс

4. **`rows.Err()` после цикла** — `Next()` не возвращает ошибку напрямую. Если он вернул `false` — мог быть конец данных или ошибка. Чтобы отличить, нужно проверить `rows.Err()`. **Это часто забывают**, и потом удивляются «странным» багам.

5. **Имя колонок в SELECT** — явно перечисляем (`SELECT id, title, done`), а не `SELECT *`. Зачем? Потому что:
   - Порядок аргументов `Scan` должен совпадать с порядком колонок
   - Если кто-то поменяет схему (добавит колонку `priority`), `SELECT *` начнёт возвращать 4 колонки, а `Scan` ожидать 3 — ошибка
   - Явные имена — это документация: читатель видит, какие данные используются

**Правило:** в Go-коде **никогда** не используй `SELECT *`. Всегда явно перечисляй колонки.

## Ядро 3: метод GetByID и sql.ErrNoRows

```go
func (r *TaskRepository) GetByID(id int64) (Task, error) {
    var t Task
    err := r.db.QueryRow(
        "SELECT id, title, done FROM tasks WHERE id = $1",
        id,
    ).Scan(&t.ID, &t.Title, &t.Done)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return Task{}, fmt.Errorf("задача не найдена: %d", id)
        }
        return Task{}, fmt.Errorf("query: %w", err)
    }

    return t, nil
}
```

Несколько новых вещей:

### Параметры запроса через $1, $2

**Никогда** не вставляй значения в SQL через `fmt.Sprintf` или конкатенацию строк:

```go
// КАТЕГОРИЧЕСКИ НЕЛЬЗЯ
sql := fmt.Sprintf("SELECT * FROM tasks WHERE id = %d", id)
```

Это **SQL injection** — самая известная уязвимость веба. Если `id` приходит от клиента, он может прислать `1; DROP TABLE tasks;--`, и ты выполнишь `DROP TABLE tasks`. Конец данным.

**Правильный способ — параметры:**

```go
db.QueryRow("SELECT * FROM tasks WHERE id = $1", id)
```

`$1`, `$2`, `$3` — это **placeholder'ы**. PostgreSQL обрабатывает их безопасно: значение **не подставляется в строку SQL**, а передаётся отдельно драйверу. Никакая инъекция невозможна, потому что значение никогда не интерпретируется как SQL.

В разных БД синтаксис placeholder'ов разный:
- **PostgreSQL**: `$1`, `$2`, `$3`
- **MySQL**: `?`, `?`, `?`
- **SQLite**: `?` или `$1`

Это одно из мест, где `database/sql` **не** даёт полной абстракции — синтаксис разный. Но идея одна: **значения отдельно от SQL**.

### sql.ErrNoRows

Когда `QueryRow` не находит ни одной строки, `Scan` возвращает специальную ошибку — **`sql.ErrNoRows`**. Это **не баг**, это нормальное поведение: ты сообщаешь, что строки нет.

Чтобы отличить «не нашли» от «упало соединение» — используется уже знакомый `errors.Is`:

```go
if errors.Is(err, sql.ErrNoRows) {
    return Task{}, fmt.Errorf("задача не найдена: %d", id)
}
```

В одном случае возвращаем «не найдено» (логическая ошибка), в другом — оборачиваем оригинальную ошибку (техническая проблема). Это позволяет вызывающему коду различать эти случаи.

## Ядро 4: метод Add через INSERT ... RETURNING

В Проектах 2-4 ты сам управлял `nextID`. Теперь — БД делает это за тебя через `BIGSERIAL`. Чтобы получить присвоенный id, используется `RETURNING`:

```go
func (r *TaskRepository) Add(title string) (Task, error) {
    var t Task
    err := r.db.QueryRow(
        "INSERT INTO tasks (title) VALUES ($1) RETURNING id, title, done",
        title,
    ).Scan(&t.ID, &t.Title, &t.Done)

    if err != nil {
        return Task{}, fmt.Errorf("insert: %w", err)
    }

    return t, nil
}
```

Несколько моментов:

1. **`INSERT ... RETURNING ...`** — расширение PostgreSQL. INSERT обычно ничего не возвращает (только `ExecResult`), но с `RETURNING` — возвращает указанные колонки.

2. **`QueryRow + Scan`** — потому что INSERT теперь работает **как** SELECT (возвращает строку).

3. **Не передаём `id`** — он будет присвоен автоматически через `BIGSERIAL`. Не передаём `done` — будет `false` (DEFAULT).

4. **Возвращаем созданную задачу целиком** — с `id`, который теперь известен. Это удобно для вызывающего кода: можно сразу показать клиенту через HTTP-ответ.

`RETURNING` — это **киллер-фича PostgreSQL**, которой нет в стандартном SQL. В MySQL пришлось бы делать INSERT, потом отдельный `SELECT LAST_INSERT_ID()` — два запроса вместо одного, и это создаёт race condition.

## Ядро 5: метод Update через db.Exec

`db.Exec` — для запросов, которые **не возвращают данных**: UPDATE, DELETE, иногда CREATE/ALTER.

```go
func (r *TaskRepository) Update(id int64, title string, done bool) (Task, error) {
    var t Task
    err := r.db.QueryRow(
        "UPDATE tasks SET title = $1, done = $2 WHERE id = $3 RETURNING id, title, done",
        title, done, id,
    ).Scan(&t.ID, &t.Title, &t.Done)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return Task{}, fmt.Errorf("задача не найдена: %d", id)
        }
        return Task{}, fmt.Errorf("update: %w", err)
    }

    return t, nil
}
```

Заметь: я снова использую **`UPDATE ... RETURNING`**. Можно было через `Exec`:

```go
result, err := r.db.Exec(
    "UPDATE tasks SET title = $1, done = $2 WHERE id = $3",
    title, done, id,
)
```

— но тогда нужно отдельно проверить `result.RowsAffected()`, чтобы понять, нашлась ли строка для обновления, и сделать второй запрос для возврата обновлённой задачи. Через `RETURNING` всё в одном запросе.

**Правило:** если запрос меняет данные **и** ты хочешь получить обратно — используй `... RETURNING`. Это чище, быстрее и атомарнее.

## Ядро 6: метод Remove через db.Exec

```go
func (r *TaskRepository) Remove(id int64) error {
    result, err := r.db.Exec(
        "DELETE FROM tasks WHERE id = $1",
        id,
    )
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

    return nil
}
```

Здесь нечего возвращать (DELETE удаляет, ничего не остаётся). Поэтому используем `Exec`. Но как понять, **была ли удалена** строка?

**`result.RowsAffected()`** возвращает количество затронутых строк. Если 0 — значит, по `WHERE id = ...` ничего не нашлось. Это и есть «задача не найдена».

Альтернатива через `RETURNING`:

```go
var deletedID int64
err := r.db.QueryRow("DELETE FROM tasks WHERE id = $1 RETURNING id", id).Scan(&deletedID)
if errors.Is(err, sql.ErrNoRows) {
    return fmt.Errorf("задача не найдена: %d", id)
}
```

Тоже работает. Какой из двух вариантов лучше — дело вкуса. `RowsAffected` чуть более идиоматично для DELETE.

## Ядро 7: контексты и QueryContext

Всё, что ты делал выше, — это «упрощённая» версия. В реальных серверах **обязательно** используют контексты:

```go
func (r *TaskRepository) GetByID(ctx context.Context, id int64) (Task, error) {
    var t Task
    err := r.db.QueryRowContext(ctx,
        "SELECT id, title, done FROM tasks WHERE id = $1",
        id,
    ).Scan(&t.ID, &t.Title, &t.Done)
    // ...
}
```

`QueryContext`, `QueryRowContext`, `ExecContext` — это версии методов с **контекстом**. Контекст несёт:

- **Дедлайн** — если запрос не уложился в N секунд, отменяется
- **Отмену** — если HTTP-клиент отвалился, мы можем отменить запрос к БД

В HTTP-обработчике у тебя уже есть контекст: `r.Context()` (где `r *http.Request`). Передаёшь его дальше:

```go
task, err := repo.GetByID(r.Context(), id)
```

**Правило для продакшена:** всегда используй `*Context` версии. **Правило для учебного модуля 4:** можешь начать без контекста и добавить в модуле 6 при интеграции с HTTP. Я покажу оба варианта.

## Ядро 8: типичные ошибки

**Забыл `defer rows.Close()` после `Query`.**
Соединение не вернётся в пул. Через несколько таких операций пул закончится, и `Query` начнёт зависать.

**Забыл `rows.Err()` после цикла.**
Не заметишь часть ошибок. `Next()` молча останавливается, а ошибка остаётся в `rows.Err()`.

**Использовал `SELECT *`.**
Связка между схемой и кодом теряется. При изменении схемы код сломается «на ровном месте».

**Конкатенация значений в SQL вместо placeholder'ов.**
SQL injection. Категорически нельзя.

**`Scan` в неправильное количество переменных.**
Запрос возвращает 3 колонки, а ты передал 2 указателя — ошибка `sql: expected 3 destination arguments in Scan, not 2`.

**`Scan` в неправильный тип.**
Колонка `BIGSERIAL` (=BIGINT), а ты передал `*int32` — может работать, может упасть. **Используй `int64` для BIGINT.**

## Полный пример репозитория

```go
package main

import (
    "database/sql"
    "errors"
    "fmt"
)

type Task struct {
    ID    int64  `json:"id"`
    Title string `json:"title"`
    Done  bool   `json:"done"`
}

type TaskRepository struct {
    db *sql.DB
}

func NewTaskRepository(db *sql.DB) *TaskRepository {
    return &TaskRepository{db: db}
}

func (r *TaskRepository) GetAll() ([]Task, error) {
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
}

func (r *TaskRepository) GetByID(id int64) (Task, error) {
    var t Task
    err := r.db.QueryRow(
        "SELECT id, title, done FROM tasks WHERE id = $1",
        id,
    ).Scan(&t.ID, &t.Title, &t.Done)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return Task{}, fmt.Errorf("задача не найдена: %d", id)
        }
        return Task{}, fmt.Errorf("query: %w", err)
    }
    return t, nil
}

func (r *TaskRepository) Add(title string) (Task, error) {
    var t Task
    err := r.db.QueryRow(
        "INSERT INTO tasks (title) VALUES ($1) RETURNING id, title, done",
        title,
    ).Scan(&t.ID, &t.Title, &t.Done)

    if err != nil {
        return Task{}, fmt.Errorf("insert: %w", err)
    }
    return t, nil
}

func (r *TaskRepository) Update(id int64, title string, done bool) (Task, error) {
    var t Task
    err := r.db.QueryRow(
        "UPDATE tasks SET title = $1, done = $2 WHERE id = $3 RETURNING id, title, done",
        title, done, id,
    ).Scan(&t.ID, &t.Title, &t.Done)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return Task{}, fmt.Errorf("задача не найдена: %d", id)
        }
        return Task{}, fmt.Errorf("update: %w", err)
    }
    return t, nil
}

func (r *TaskRepository) Remove(id int64) error {
    result, err := r.db.Exec(
        "DELETE FROM tasks WHERE id = $1",
        id,
    )
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

    return nil
}
```

## Вопросы на понимание

1. Чем `db.Query` отличается от `db.QueryRow`?
2. Почему **обязательно** делать `defer rows.Close()` после успешного `Query`?
3. Зачем нужен `rows.Err()` после цикла?
4. Почему мы **никогда** не используем `SELECT *` в Go-коде?
5. Что такое `$1`, `$2` в запросах? Зачем они?
6. Что такое **SQL injection** и как от неё защищаться?
7. Что такое `sql.ErrNoRows` и как с ней работать?
8. Что такое `RETURNING` в PostgreSQL? Какие у него преимущества?
9. Чем `db.Exec` отличается от `db.Query`?
10. Что такое `RowsAffected` и зачем оно?
