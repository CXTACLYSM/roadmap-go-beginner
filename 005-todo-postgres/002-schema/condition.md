# Модуль 2. Схема и миграции — условие

## Задание

Создать в проекте папку `migrations/` с пронумерованными `.sql` файлами. Применить их к свежей БД. Проверить, что схема воспроизводится с нуля.

## Что должно произойти

```bash
$ tree migrations/
migrations/
├── 001_create_tasks_table.sql
└── 002_index_done.sql

$ docker rm -f todo-pg
$ docker volume rm todo-pg-data
$ docker run -d --name todo-pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -v todo-pg-data:/var/lib/postgresql/data postgres:16
$ sleep 3   # ждём, пока БД поднимется

$ export PGPASSWORD=secret
$ psql -h localhost -U postgres -d postgres -f migrations/001_create_tasks_table.sql
CREATE TABLE
$ psql -h localhost -U postgres -d postgres -f migrations/002_index_done.sql
CREATE INDEX

$ psql -h localhost -U postgres -d postgres -c "\d tasks"
                                  Table "public.tasks"
   Column    |           Type           | Collation | Nullable |              Default              
-------------+--------------------------+-----------+----------+-----------------------------------
 id          | bigint                   |           | not null | nextval('tasks_id_seq'::regclass)
 title       | text                     |           | not null | 
 done        | boolean                  |           | not null | false
 created_at  | timestamp with time zone |           | not null | now()
Indexes:
    "tasks_pkey" PRIMARY KEY, btree (id)
    "idx_tasks_done" btree (done)
```

## Требования

1. Создана папка `migrations/` в корне `005-todo-postgres/`.
2. Создан файл `migrations/001_create_tasks_table.sql`:
   ```sql
   CREATE TABLE tasks (
       id         BIGSERIAL PRIMARY KEY,
       title      TEXT NOT NULL,
       done       BOOLEAN NOT NULL DEFAULT false,
       created_at TIMESTAMPTZ NOT NULL DEFAULT now()
   );
   ```
3. Создан файл `migrations/002_index_done.sql`:
   ```sql
   CREATE INDEX idx_tasks_done ON tasks (done);
   ```
4. **Имена файлов** начинаются с трёхзначного номера и содержат осмысленное описание.
5. Контейнер пересоздан **с чистым volume** (потеря старых данных).
6. Обе миграции применены через `psql -f`.
7. `\d tasks` показывает все 4 колонки с правильными типами.
8. `\d tasks` показывает **два индекса** — `tasks_pkey` (автоматический от PRIMARY KEY) и `idx_tasks_done` (наш).
9. Сделано несколько `INSERT` для проверки, что `created_at` заполняется автоматически.
10. Файлы миграций добавлены в git (это часть проекта!).

## Шаги

1. В корне `005-todo-postgres/` создай папку `migrations`
2. Создай файл `001_create_tasks_table.sql` с DDL для таблицы
3. Создай файл `002_index_done.sql` с CREATE INDEX
4. Удали и пересоздай контейнер с чистым volume:
   ```bash
   docker rm -f todo-pg
   docker volume rm todo-pg-data
   docker run -d --name todo-pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -v todo-pg-data:/var/lib/postgresql/data postgres:16
   sleep 3
   ```
5. Примени миграции по порядку:
   ```bash
   export PGPASSWORD=secret
   psql -h localhost -U postgres -d postgres -f migrations/001_create_tasks_table.sql
   psql -h localhost -U postgres -d postgres -f migrations/002_index_done.sql
   ```
6. Проверь схему через `psql ... -c "\d tasks"`
7. Сделай 2-3 INSERT, проверь, что `created_at` автоматически заполнен
8. Закоммить миграции в git

## Эксперименты

1. **Применённая миграция дважды:** запусти `psql -f migrations/001_create_tasks_table.sql` ещё раз. Что произошло?
   ```
   ERROR: relation "tasks" already exists
   ```
   Это **не баг**, это правильное поведение. Миграции **не идемпотентны** по умолчанию — повторное применение упадёт. В продакшен-инструментах (golang-migrate и т.д.) есть механизм отслеживания применённых миграций, который решает эту проблему. Сейчас — просто не запускай дважды.

2. **Idempotent миграция через `IF NOT EXISTS`:**
   ```sql
   CREATE TABLE IF NOT EXISTS tasks (
       id BIGSERIAL PRIMARY KEY,
       ...
   );
   ```
   Теперь миграцию можно запускать сколько угодно раз. **Но это уловка** — лучше отслеживать применённые миграции через инструмент, чем «защищаться» через `IF NOT EXISTS`.

3. **Третья миграция — добавить колонку:**
   ```sql
   -- migrations/003_add_priority.sql
   ALTER TABLE tasks ADD COLUMN priority INTEGER NOT NULL DEFAULT 0;
   ```
   Примени, проверь через `\d tasks`. Сделай SELECT — у всех существующих строк `priority = 0` (благодаря DEFAULT).

4. **Удали колонку (опасно!):**
   ```sql
   -- migrations/004_remove_priority.sql
   ALTER TABLE tasks DROP COLUMN priority;
   ```
   Все данные в этой колонке потеряны. Это работает мгновенно (в нашем случае), но на больших таблицах может занимать минуты.

5. **Сделай попытку добавить NOT NULL колонку без DEFAULT:**
   ```sql
   ALTER TABLE tasks ADD COLUMN tag TEXT NOT NULL;
   ```
   Получишь ошибку: `column "tag" of relation "tasks" contains null values`. PostgreSQL не может присвоить NULL новой обязательной колонке. Решение — сначала добавить с DEFAULT, потом, если нужно, убрать DEFAULT через ещё одну ALTER.

6. **Проверь индексы под нагрузкой (опционально):**
   ```sql
   -- Создаём 100k задач для теста
   INSERT INTO tasks (title, done) 
   SELECT 'task ' || i, i % 2 = 0
   FROM generate_series(1, 100000) i;

   -- С индексом
   EXPLAIN ANALYZE SELECT * FROM tasks WHERE done = false LIMIT 10;
   ```
   Посмотри на план. Должен быть `Index Scan using idx_tasks_done`. Без индекса было бы `Seq Scan` (полный перебор). Это и есть смысл индексов.

7. **Проверь автоматическое заполнение `created_at`:**
   ```sql
   INSERT INTO tasks (title) VALUES ('test');
   SELECT id, title, created_at FROM tasks ORDER BY id DESC LIMIT 1;
   ```
   `created_at` будет заполнено текущим временем UTC (с часовым поясом).

8. **Сравни TIMESTAMP и TIMESTAMPTZ:**
   ```sql
   CREATE TABLE test_time (
       a TIMESTAMP DEFAULT now(),
       b TIMESTAMPTZ DEFAULT now()
   );
   INSERT INTO test_time DEFAULT VALUES;
   SELECT * FROM test_time;
   ```
   `b` (TIMESTAMPTZ) покажется с указанием часового пояса. `a` (TIMESTAMP) — без. В Go это будет важно: `time.Time` всегда имеет часовой пояс, и `TIMESTAMPTZ` — естественный маппинг.

## Что показать Claude

Покажи Claude свою таблицу `migrations/` и спроси: «как бы я добавил миграцию, которая ставит NOT NULL на существующую колонку, в которой могут быть NULL?». Это **важный вопрос продакшен-миграций**, и правильный ответ — **двумя шагами**:

1. Сначала миграция, которая обновляет все существующие NULL на какое-то значение (например, `''` или `0`)
2. Потом миграция, которая ставит NOT NULL

Если попробовать одной миграцией — упадёт. Это **классический паттерн** «многошаговых миграций», и его нужно знать. В Проекте 7 ты увидишь это в полный рост, когда будешь добавлять авторизацию к существующей таблице.

Запиши себе: **миграции всегда односторонние** (forward-only), **никогда не правь старые**, и **думай о том, что в таблице уже есть данные**, а не только о новой схеме.
