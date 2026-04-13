# Модуль 1. Установка и базовый SQL — условие

## Задание

Установить PostgreSQL через Docker, подключиться через `psql`, создать таблицу `tasks` и **вручную** прогнать все CRUD-операции на SQL. Никакого Go в этом модуле нет — только командная строка и SQL.

## Что должно произойти

```bash
$ docker run -d --name todo-pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -v todo-pg-data:/var/lib/postgresql/data postgres:16

$ docker ps
CONTAINER ID   IMAGE         ...   NAMES
abc123         postgres:16   ...   todo-pg

$ psql -h localhost -U postgres -d postgres
Password: secret
postgres=# CREATE TABLE tasks (...);
CREATE TABLE
postgres=# INSERT INTO tasks (title) VALUES ('Купить молоко');
INSERT 0 1
postgres=# SELECT * FROM tasks;
 id |     title      | done
----+----------------+------
  1 | Купить молоко  | f
(1 row)
postgres=# \q
```

## Требования

1. Docker установлен и работает (`docker --version`)
2. `psql` установлен и работает (`psql --version`)
3. Запущен контейнер PostgreSQL версии 16 с именем `todo-pg`
4. Контейнер использует **named volume** (`-v todo-pg-data:/var/lib/postgresql/data`)
5. Подключение к PostgreSQL через `psql` работает
6. Создана таблица `tasks` со схемой:
   ```sql
   CREATE TABLE tasks (
       id      SERIAL PRIMARY KEY,
       title   TEXT NOT NULL,
       done    BOOLEAN NOT NULL DEFAULT false
   );
   ```
7. Через `psql` вручную выполнены **все** базовые операции:
   - **3 INSERT** с разными задачами
   - **SELECT \*** — увидеть все
   - **SELECT с WHERE** — найти одну по id
   - **SELECT с WHERE done = false** — найти невыполненные
   - **UPDATE** одной задачи (поставить `done = true`)
   - **DELETE** одной задачи
   - **SELECT** в конце для проверки
8. Использована `\dt` для проверки списка таблиц
9. Использована `\d tasks` для просмотра структуры таблицы
10. Контейнер пережил **остановку и запуск** (`docker stop` → `docker start`)

## Шаги

### Установка

1. Установи Docker, если ещё нет:
   - **Linux**: следуй [официальной инструкции](https://docs.docker.com/engine/install/)
   - **macOS**: [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/)
   - **Windows**: Docker Desktop через WSL 2

2. Установи `psql`:
   - **Linux**: `sudo apt install postgresql-client`
   - **macOS**: `brew install libpq && brew link --force libpq`
   - **Windows**: ставится с PostgreSQL Installer

3. Проверь:
   ```bash
   docker --version
   psql --version
   ```

### Запуск БД

4. Создай и запусти контейнер:
   ```bash
   docker run -d \
       --name todo-pg \
       -e POSTGRES_PASSWORD=secret \
       -p 5432:5432 \
       -v todo-pg-data:/var/lib/postgresql/data \
       postgres:16
   ```

5. Проверь, что запустился:
   ```bash
   docker ps
   ```
   Должна быть строка с `todo-pg` и статусом `Up`.

6. Посмотри логи (опционально):
   ```bash
   docker logs todo-pg
   ```
   В конце должно быть что-то вроде `database system is ready to accept connections`.

### Подключение и работа

7. Подключись через psql:
   ```bash
   psql -h localhost -U postgres -d postgres
   ```
   Введи пароль `secret`.

8. Создай таблицу (вставь SQL из требования 6).

9. Проверь структуру:
   ```
   \dt
   \d tasks
   ```

10. Вставь три задачи:
    ```sql
    INSERT INTO tasks (title) VALUES ('Купить молоко');
    INSERT INTO tasks (title) VALUES ('Позвонить маме');
    INSERT INTO tasks (title) VALUES ('Сделать домашку');
    ```

11. Прогон всех SELECT, UPDATE, DELETE из требований.

12. Выйди (`\q`).

### Тест выживания

13. Останови контейнер:
    ```bash
    docker stop todo-pg
    ```

14. Запусти снова:
    ```bash
    docker start todo-pg
    ```

15. Подключись через psql, сделай `SELECT * FROM tasks;`. Данные должны быть на месте — это работа volume.

## Эксперименты

1. **Попробуй INSERT с указанным ID:**
   ```sql
   INSERT INTO tasks (id, title) VALUES (100, 'тест с id');
   SELECT * FROM tasks;
   ```
   Работает, и теперь у тебя `id = 100`. Что произойдёт со следующим обычным INSERT? Сделай `INSERT INTO tasks (title) VALUES ('что дальше');` — какой `id`? Возможно, это будет **не 101** — это особенность работы `SERIAL` (sequences).

2. **Сделай UPDATE без WHERE** (на тестовых данных):
   ```sql
   UPDATE tasks SET done = true;
   SELECT * FROM tasks;
   ```
   Все задачи стали выполненными. **Это та самая катастрофа**, которой надо избегать в продакшене. Верни как было через ещё один UPDATE:
   ```sql
   UPDATE tasks SET done = false WHERE id < 100;
   ```

3. **Попробуй вставить NULL в title:**
   ```sql
   INSERT INTO tasks (title) VALUES (NULL);
   ```
   Получишь ошибку: `null value in column "title" of relation "tasks" violates not-null constraint`. Это работа `NOT NULL` — БД защищает тебя от пустых значений в обязательной колонке.

4. **Подсчёт строк:**
   ```sql
   SELECT COUNT(*) FROM tasks;
   SELECT COUNT(*) FROM tasks WHERE done = true;
   SELECT COUNT(*) FROM tasks WHERE done = false;
   ```
   Заметь, как одной строкой получается то, что в Go было бы `for + counter`.

5. **Сортировка:**
   ```sql
   SELECT * FROM tasks ORDER BY id DESC;
   SELECT * FROM tasks ORDER BY title;
   SELECT * FROM tasks ORDER BY done, id;
   ```
   `ORDER BY done, id` — сначала по `done`, потом (внутри одинакового done) по `id`. Это многоуровневая сортировка.

6. **LIMIT и OFFSET:**
   ```sql
   SELECT * FROM tasks LIMIT 2;
   SELECT * FROM tasks LIMIT 2 OFFSET 1;  -- пропусти первую, дай 2
   ```
   Это основа **пагинации** — когда ты возвращаешь данные «по страницам».

7. **Используй `RETURNING`:**
   ```sql
   INSERT INTO tasks (title) VALUES ('новая') RETURNING id;
   INSERT INTO tasks (title) VALUES ('ещё новая') RETURNING *;
   ```
   В первом случае получишь только новый id, во втором — всю строку. Это незаменимо при создании задач из Go (модуль 4).

8. **Удали и пересоздай контейнер:**
   ```bash
   docker rm -f todo-pg
   docker run -d --name todo-pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -v todo-pg-data:/var/lib/postgresql/data postgres:16
   ```
   Подключись, проверь `\dt`. Таблица **на месте**, потому что данные хранятся в volume `todo-pg-data`. Это и есть сила именованных volume'ов.

9. **(Опасный) Удали volume:**
   ```bash
   docker rm -f todo-pg
   docker volume rm todo-pg-data
   docker run -d --name todo-pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -v todo-pg-data:/var/lib/postgresql/data postgres:16
   ```
   Теперь `\dt` пуст. Это потеря всех данных. Из этого вынеси урок: **бэкапы**. В продакшене они обязательны, в учебном проекте — не критично, но осознавай.

## Что показать Claude

Покажи Claude свой полный лог сессии в `psql` (хотя бы из памяти, по описанию). Спроси: «правильно ли я понял, чем `TEXT` отличается от `VARCHAR(N)`? Когда выбирать что?». Это популярный вопрос новичка, и ответ — `TEXT` в PostgreSQL почти всегда лучше: нет ограничения на длину, нет проверки длины на каждой вставке, и для PostgreSQL они хранятся одинаково. `VARCHAR(N)` имеет смысл только если ты хочешь явно ограничить длину на уровне БД.

Запиши себе: **используй `TEXT` по умолчанию**, `VARCHAR(N)` только при сознательной необходимости.
