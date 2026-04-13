# Модуль 4. Посты с владельцем — условие

## Задание

Создать таблицу `posts` со связью на `users` через FK с CASCADE. Реализовать `PostRepository` с CRUD-методами, JOIN для загрузки автора. Реализовать 5 эндпоинтов с **проверкой владения** для PUT/DELETE.

## Что должно произойти

```bash
# Регистрация и логин (из модуля 3)
$ TOKEN1=$(curl -s -X POST http://localhost:8080/login -H "Content-Type: application/json" \
    -d '{"email":"lev@example.com","password":"strong-password-123"}' | jq -r .token)

# Создание поста (требует токен)
$ curl -X POST http://localhost:8080/posts \
    -H "Authorization: Bearer $TOKEN1" \
    -H "Content-Type: application/json" \
    -d '{"title":"Привет, мир","body":"Это мой первый пост"}'
{"id":1,"author_id":1,"title":"Привет, мир","body":"Это мой первый пост","created_at":"...","updated_at":"..."}

# Список постов (публично)
$ curl http://localhost:8080/posts
[{"id":1,"author_id":1,"title":"...","body":"...","created_at":"...","updated_at":"...","author":{"id":1,"email":"lev@example.com","created_at":"..."}}]

# Один пост с автором
$ curl http://localhost:8080/posts/1
{"id":1,"author_id":1,...,"author":{"id":1,"email":"lev@example.com",...}}

# Создание без токена
$ curl -i -X POST http://localhost:8080/posts -H "Content-Type: application/json" -d '{"title":"x","body":"y"}'
HTTP/1.1 401 Unauthorized
{"error":"missing authorization header"}

# Обновление своего поста
$ curl -X PUT http://localhost:8080/posts/1 \
    -H "Authorization: Bearer $TOKEN1" \
    -H "Content-Type: application/json" \
    -d '{"title":"Обновлённое","body":"Новое тело"}'
{"id":1,"title":"Обновлённое",...}

# Регистрация второго пользователя
$ curl -X POST http://localhost:8080/register -d '{"email":"anya@example.com","password":"another-pass-123"}' -H "Content-Type: application/json"
$ TOKEN2=$(curl -s -X POST http://localhost:8080/login -H "Content-Type: application/json" \
    -d '{"email":"anya@example.com","password":"another-pass-123"}' | jq -r .token)

# Попытка Ани отредактировать пост Льва
$ curl -i -X PUT http://localhost:8080/posts/1 \
    -H "Authorization: Bearer $TOKEN2" \
    -H "Content-Type: application/json" \
    -d '{"title":"hack","body":"hack"}'
HTTP/1.1 403 Forbidden
{"error":"forbidden: not the author"}

# Удаление своего поста
$ curl -i -X DELETE http://localhost:8080/posts/1 -H "Authorization: Bearer $TOKEN1"
HTTP/1.1 204 No Content

# Попытка получить удалённый
$ curl -i http://localhost:8080/posts/1
HTTP/1.1 404 Not Found
{"error":"post not found"}
```

## Требования

### Миграция

1. Создан файл `migrations/002_create_posts.sql`:
   ```sql
   CREATE TABLE posts (
       id         BIGSERIAL PRIMARY KEY,
       author_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
       title      TEXT NOT NULL CHECK (length(title) > 0 AND length(title) <= 200),
       body       TEXT NOT NULL CHECK (length(body) > 0),
       created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
       updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
   );
   CREATE INDEX idx_posts_author_id ON posts (author_id);
   CREATE INDEX idx_posts_created_at ON posts (created_at DESC);
   ```
2. Миграция применена

### Domain

3. Создан тип `Post` в `internal/domain/post.go` с полями `ID`, `AuthorID`, `Title`, `Body`, `CreatedAt`, `UpdatedAt` и JSON-тегами
4. Создан тип `PostWithAuthor` через **embedded struct** `Post` + поле `Author User`

### Storage

5. Создан тип `PostRepository` с полем `db *sql.DB`
6. Реализован конструктор `NewPostRepository`
7. Объявлена `ErrPostNotFound`
8. Реализованы методы:
   - `Create(ctx, authorID, title, body) (*domain.Post, error)`
   - `GetByID(ctx, id) (*domain.PostWithAuthor, error)` — **с JOIN на users**
   - `List(ctx) ([]domain.PostWithAuthor, error)` — **с JOIN, сортировка `ORDER BY created_at DESC`**
   - `Update(ctx, id, title, body) (*domain.Post, error)` — **обновляет `updated_at = now()`**
   - `Delete(ctx, id) error`
9. **Все запросы используют `Context` методы**
10. **`Update` НЕ проверяет владение** — это бизнес-правило, оно в handler
11. `GetByID` и `Update` возвращают `ErrPostNotFound` при `sql.ErrNoRows`
12. `Delete` использует `RowsAffected` для обнаружения «не найдено»
13. `List` возвращает `[]domain.PostWithAuthor{}` (пустой слайс), не `nil`

### API

14. Добавлено поле `posts *storage.PostRepository` в `Handler`
15. Обновлён `NewHandler` принимает посты
16. Реализованы handlers в `internal/api/posts.go`:
    - `listPosts` — публичный
    - `getPost` — публичный
    - `createPost` — защищённый, валидация title/body
    - `updatePost` — защищённый, **проверка владения**
    - `deletePost` — защищённый, **проверка владения**
17. Helper `parseInt64(w, r, name) (int64, bool)` для парсинга id из path

### Проверка владения

18. `updatePost` сначала вызывает `GetByID`, проверяет `existing.Post.AuthorID == userID`
19. Если не совпадает → **403 Forbidden** с сообщением `"forbidden: not the author"`
20. То же самое в `deletePost`

### Регистрация роутов

21. `GET /posts` — публичный
22. `GET /posts/{id}` — публичный
23. `POST /posts` — через middleware
24. `PUT /posts/{id}` — через middleware
25. `DELETE /posts/{id}` — через middleware

### Main

26. В `main.go` создаётся `postRepo := storage.NewPostRepository(db)`
27. Передаётся в `NewHandler`

### Поведение

28. Все сценарии из примера работают
29. Эндпоинты модулей 1-3 продолжают работать
30. **Удаление пользователя через `psql`** автоматически удаляет его посты (CASCADE)

## Шаги

1. Создай миграцию `002_create_posts.sql` и примени
2. Создай `internal/domain/post.go` с `Post` и `PostWithAuthor`
3. Создай `internal/storage/post.go` с `PostRepository` и всеми методами
4. Создай `internal/api/posts.go` с handlers
5. Helper `parseInt64` положи рядом (или в отдельный файл `internal/api/helpers.go`)
6. Обнови `Handler` — добавь поле `posts`
7. Обнови `NewHandler` — принимает `posts`
8. Обнови `RegisterRoutes` — добавь 5 эндпоинтов
9. Обнови `main.go` — создай `postRepo`, передай в handler
10. Запусти, прогон полного сценария

## Эксперименты

1. **Удали пользователя через psql, посмотри на посты:**
   ```sql
   INSERT INTO users (email, password_hash) VALUES ('test@x.com', '$2a$10$fake') RETURNING id;
   -- допустим, id = 5
   INSERT INTO posts (author_id, title, body) VALUES (5, 'test', 'test');
   SELECT count(*) FROM posts WHERE author_id = 5;  -- 1
   DELETE FROM users WHERE id = 5;
   SELECT count(*) FROM posts WHERE author_id = 5;  -- 0!
   ```
   Это работа `ON DELETE CASCADE`.

2. **Попробуй вставить пост с несуществующим author_id:**
   ```sql
   INSERT INTO posts (author_id, title, body) VALUES (9999, 'test', 'test');
   ```
   Получишь `insert or update on table "posts" violates foreign key constraint`. Это защита FK.

3. **Попробуй создать пост с пустым title:**
   ```bash
   curl -X POST http://localhost:8080/posts -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"title":"","body":"x"}'
   ```
   400 от твоей валидации в Go. **Также** — попробуй обойти Go-валидацию и отправить через `psql`:
   ```sql
   INSERT INTO posts (author_id, title, body) VALUES (1, '', 'test');
   ```
   PostgreSQL отклонит из-за `CHECK (length(title) > 0)`. **Двойная защита**.

4. **Попробуй очень длинный title:**
   ```bash
   TITLE=$(python3 -c "print('x' * 250)")
   curl -X POST http://localhost:8080/posts -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"title\":\"$TITLE\",\"body\":\"x\"}"
   ```
   400 от Go (`title too long`). И БД тоже бы отклонила.

5. **EXPLAIN на SELECT с фильтром по author_id:**
   ```sql
   EXPLAIN ANALYZE SELECT * FROM posts WHERE author_id = 1;
   ```
   Должен использовать `idx_posts_author_id`. Если бы индекса не было — был бы Sequential Scan.

6. **Сравни production-вариант UPDATE с проверкой в WHERE:** временно перепиши `updatePost` так:
   ```go
   // Один запрос: либо обновляет, либо нет
   result, err := r.db.ExecContext(ctx, `
       UPDATE posts SET title = $1, body = $2, updated_at = now()
       WHERE id = $3 AND author_id = $4
   `, title, body, id, userID)
   // Если RowsAffected == 0 → либо нет поста, либо не твой
   ```
   Это **более эффективно** (один запрос вместо двух) и **избегает race condition**. Минус — нельзя различить «нет поста» и «не твой» (всегда вернёт что-то общее). Для production — это **правильнее**.

7. **Создай два поста, проверь сортировку:** список должен идти от **новых к старым** благодаря `ORDER BY created_at DESC`.

8. **Сделай POST с невалидным JSON:**
   ```bash
   curl -X POST http://localhost:8080/posts -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d 'not json'
   ```
   400 от твоей валидации. Прежде, чем код доходит до БД.

9. **Сравни время `GET /posts` (с JOIN) и hypothetical `GET /posts` (без JOIN, с автором как `null`):**
   - С JOIN: один запрос, чуть медленнее
   - Без JOIN: один запрос, быстрее, но клиент не получает имя автора

   Для блога JOIN оправдан. Для high-throughput сервиса — могут денормализовать (хранить копию `author_email` в `posts`).

## Что показать Claude

Покажи Claude свой `updatePost` и спроси: «правильно ли я разделил проверку владения и обновление? Какие плюсы и минусы у двух-шагового подхода vs одношагового через WHERE?». Это важный вопрос:

- **Двухшаговый (наш)** — `GetByID` + проверка + `Update`. **Плюс**: легче отличать ошибки (404 vs 403). **Минус**: race condition между шагами; два запроса вместо одного.

- **Одношаговый** — `UPDATE ... WHERE id = $1 AND author_id = $2 RETURNING ...`. **Плюс**: атомарно, один запрос. **Минус**: труднее различить «нет» vs «не твой».

В учебном модуле выбран двухшаговый ради ясности. В production обычно одношаговый.

Запиши себе на будущее: **в реальных системах атомарность важнее красоты**. Если можешь сделать одним запросом — делай одним. Race condition — это категория багов, которая воспроизводится только под нагрузкой и которая дорога в отладке.
