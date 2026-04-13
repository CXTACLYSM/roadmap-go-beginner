# Модуль 5. Транзакции — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какой метод `*sql.DB` начинает транзакцию?
2. Какой тип возвращает `Begin()`?
3. Какие два метода завершают транзакцию?
4. Что нужно сделать **сразу** после `Begin`?
5. Как сделать INSERT внутри транзакции — через `db` или через `tx`?

## Уровень 2: про db.Begin и Tx

```go
tx, err := r.db.Begin()
if err != nil {
    return nil, fmt.Errorf("begin: %w", err)
}
defer tx.Rollback()
```

`*sql.Tx` — это объект транзакции. Он реализует те же методы, что `*sql.DB`:

- `tx.Query(...)` — запрос с многими строками
- `tx.QueryRow(...)` — запрос с одной строкой
- `tx.Exec(...)` — запрос без результата

Все эти методы работают **внутри одной транзакции**. Никаких новых соединений не открывается, всё через одно.

## Уровень 3: про defer Rollback

**Сразу** после `Begin` — `defer tx.Rollback()`. Без исключений.

```go
tx, err := r.db.Begin()
if err != nil {
    return nil, err
}
defer tx.Rollback()  // ← вот эта строка

// ... остальной код ...

if err := tx.Commit(); err != nil {
    return nil, err
}
return result, nil
```

Что происходит:

- **Успех**: `Commit()` срабатывает, потом `defer Rollback()` срабатывает на выходе, но транзакция уже закрыта — `Rollback` молча возвращает ошибку «уже закрыта», которую мы игнорируем (defer ignore-ит возвраты).
- **Ошибка до Commit**: функция выходит через `return err`. `defer Rollback()` срабатывает, отменяет транзакцию.
- **Panic**: `defer Rollback()` срабатывает даже при panic. Транзакция отменяется.

## Уровень 4: про tx.Exec vs db.Exec

Внутри транзакции **всегда** через `tx`:

```go
tx, _ := r.db.Begin()
defer tx.Rollback()

// ПРАВИЛЬНО — через tx
tx.Exec("INSERT INTO tasks (title) VALUES ($1)", title)

// НЕПРАВИЛЬНО — через r.db (выполнится вне транзакции!)
r.db.Exec("INSERT INTO tasks (title) VALUES ($1)", title)
```

Это типичная ошибка новичка. **`tx` и `db` — это две разные вещи**, даже если они выглядят похоже. Запрос через `db` выполнится **в другом соединении из пула**, вне твоей транзакции.

## Уровень 5: про шаблон AddBatch

```go
func (r *TaskRepository) AddBatch(titles []string) ([]Task, error) {
    if len(titles) == 0 {
        return nil, nil
    }

    tx, err := r.db.Begin()
    if err != nil {
        return nil, fmt.Errorf("begin: %w", err)
    }
    defer tx.Rollback()

    created := make([]Task, 0, len(titles))
    for _, title := range titles {
        var t Task
        err := tx.QueryRow(
            "INSERT INTO tasks (title) VALUES ($1) RETURNING id, title, done",
            title,
        ).Scan(&t.ID, &t.Title, &t.Done)
        if err != nil {
            return nil, fmt.Errorf("insert %q: %w", title, err)
            // defer Rollback откатит транзакцию
        }
        created = append(created, t)
    }

    if err := tx.Commit(); err != nil {
        return nil, fmt.Errorf("commit: %w", err)
    }

    return created, nil
}
```

Заметь:
- **`make([]Task, 0, len(titles))`** — слайс с **нулевой длиной**, но **ёмкостью** под все задачи. Это микро-оптимизация: `append` не будет переаллоцировать.
- **Одна функция** содержит: Begin, defer, цикл с операциями, Commit. Это «единица работы».

## Уровень 6: про CHECK constraint

Чтобы испытать `Rollback` на сбое, нужна операция, которая **гарантированно** упадёт. Самый простой способ — добавить ограничение в БД:

```sql
ALTER TABLE tasks ADD CONSTRAINT title_not_empty CHECK (length(title) > 0);
```

Теперь вставка `INSERT INTO tasks (title) VALUES ('')` упадёт с ошибкой:

```
ERROR: new row for relation "tasks" violates check constraint "title_not_empty"
```

И эта ошибка пробрасывается через `tx.Exec` → твой Go-код увидит `error`, и `defer Rollback` сработает.

## Уровень 7: про сценарий с ошибкой в main

```go
log.Println("--- Эксперимент: сбой посередине ---")
_, err := repo.AddBatch([]string{"задача 1", "", "задача 3"})
if err != nil {
    log.Printf("AddBatch с одной плохой задачей упал: %v", err)
}

tasks, _ := repo.GetAll()
log.Printf("После сбоя: %d задач (ничего не добавилось)", len(tasks))
```

Если у тебя `Add` не падает на пустом title — вспомни, что нужно **сначала применить миграцию с CHECK constraint**. Без неё PostgreSQL разрешает пустые строки.

## Уровень 8: про возврат ошибки и Rollback

Обрати внимание: после `return ..., err` `defer Rollback` срабатывает **автоматически**. Не нужно явно писать:

```go
// ИЗБЫТОЧНО
if err != nil {
    tx.Rollback()
    return nil, err
}

// ПРАВИЛЬНО — defer всё сделает
if err != nil {
    return nil, err
}
```

`defer` гарантирует вызов на любом пути выхода из функции. Это и есть его сила.

## Уровень 9: про Commit и его ошибку

```go
if err := tx.Commit(); err != nil {
    return nil, fmt.Errorf("commit: %w", err)
}
```

`Commit` тоже может упасть — например, из-за нарушения constraint, который проверяется отложенно. В этом случае:

- Изменения **не применились**
- `defer Rollback` сработает, попытается откатить — но транзакция уже не в открытом состоянии (это после неудачного Commit). `Rollback` молча вернёт ошибку, которую мы игнорируем.

В целом ошибки `Commit` встречаются редко. Самые частые причины — `serialization_failure` (при `SERIALIZABLE` уровне изоляции) или `deadlock_detected`. Для нашего проекта это невероятно.

## Если совсем тупик

Скажи Claude конкретно: «модуль 5 проекта 5, [конкретная проблема]». Самые частые ситуации:

- **«После сбойного AddBatch в БД появились задачи»** → используешь `r.db` вместо `tx` внутри цикла
- **«AddBatch не падает на пустом title»** → не применил миграцию с CHECK constraint
- **«После успешного AddBatch в БД ничего нет»** → забыл `tx.Commit()`
- **«error: tx already committed»** → пытаешься использовать tx после Commit
