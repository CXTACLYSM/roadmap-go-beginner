# Модуль 5. Комментарии и JOIN — условие

## Задание

Создать таблицу `comments` с двумя FK (на posts и users), реализовать `CommentRepository`, добавить два эндпоинта `GET /posts/{id}/comments` и `POST /posts/{id}/comments`. Применить sub-resource паттерн.

## Что должно произойти

```bash
# Создаём пост (модуль 4)
$ TOKEN=$(curl -s -X POST http://localhost:8080/login -H "Content-Type: application/json" -d '{"email":"lev@example.com","password":"strong-password-123"}' | jq -r .token)
$ curl -X POST http://localhost:8080/posts -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"title":"Привет","body":"Это пост"}'
# {"id":1,...}

# Список комментариев к посту (пусто)
$ curl http://localhost:8080/posts/1/comments
[]

# Добавляем комментарий
$ curl -X POST http://localhost:8080/posts/1/comments \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"body":"Круто!"}'
{"id":1,"post_id":1,"author_id":1,"body":"Круто!","created_at":"..."}

# Регистрация второго пользователя и комментарий от него
$ TOKEN2=$(curl ... вход Ани)
$ curl -X POST http://localhost:8080/posts/1/comments \
    -H "Authorization: Bearer $TOKEN2" \
    -H "Content-Type: application/json" \
    -d '{"body":"Согласна!"}'
{"id":2,...}

# Список комментариев — два, с авторами
$ curl http://localhost:8080/posts/1/comments
[
  {"id":1,"post_id":1,"author_id":1,"body":"Круто!","created_at":"...","author":{"id":1,"email":"lev@example.com","created_at":"..."}},
  {"id":2,"post_id":1,"author_id":2,"body":"Согласна!","created_at":"...","author":{"id":2,"email":"anya@example.com","created_at":"..."}}
]

# Комментарий к несуществующему посту
$ curl -i -X POST http://localhost:8080/posts/9999/comments \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"body":"x"}'
HTTP/1.1 404 Not Found
{"error":"post not found"}

# Список комментариев к несуществующему посту
$ curl -i http://localhost:8080/posts/9999/comments
HTTP/1.1 404 Not Found
{"error":"post not found"}

# Создание без токена
$ curl -i -X POST http://localhost:8080/posts/1/comments -H "Content-Type: application/json" -d '{"body":"x"}'
HTTP/1.1 401 Unauthorized

# Удаление поста удаляет его комментарии (CASCADE)
$ curl -X DELETE http://localhost:8080/posts/1 -H "Authorization: Bearer $TOKEN"
$ psql ... -c "SELECT count(*) FROM comments WHERE post_id = 1;"
# 0
```

## Требования

### Миграция

1. Создан `migrations/003_create_comments.sql`:
   ```sql
   CREATE TABLE comments (
       id         BIGSERIAL PRIMARY KEY,
       post_id    BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
       author_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
       body       TEXT NOT NULL CHECK (length(body) > 0 AND length(body) <= 2000),
       created_at TIMESTAMPTZ NOT NULL DEFAULT now()
   );
   CREATE INDEX idx_comments_post_id ON comments (post_id);
   CREATE INDEX idx_comments_author_id ON comments (author_id);
   ```
2. Миграция применена

### Domain

3. Создан тип `Comment` в `internal/domain/comment.go` с полями `ID`, `PostID`, `AuthorID`, `Body`, `CreatedAt`
4. Создан тип `CommentWithAuthor` через embedded `Comment` + `Author User`

### Storage

5. Создан тип `CommentRepository` с полем `db *sql.DB`
6. Реализован конструктор `NewCommentRepository`
7. Объявлена `ErrCommentNotFound` (опционально, для будущего)
8. Реализована функция **`isForeignKeyViolation(err) bool`** в `storage/errors.go`
9. Реализованы методы:
   - `Create(ctx, postID, authorID, body) (*domain.Comment, error)`
   - `ListByPost(ctx, postID) ([]domain.CommentWithAuthor, error)` — **с JOIN** на users, **сортировка ASC**
10. **`Create` ловит FK violation** и возвращает `ErrPostNotFound`
11. Все методы используют `*Context`
12. `ListByPost` возвращает `[]domain.CommentWithAuthor{}` для пустого случая

### API

13. Тип `Handler` имеет поле `comments *storage.CommentRepository`
14. `NewHandler` принимает `comments`
15. Создан `internal/api/comments.go`
16. Реализован `listComments(w, r)`:
    - Парсит `id` (это id поста)
    - **Сначала проверяет существование поста** через `posts.GetByID`
    - Если поста нет → 404
    - Если есть — возвращает список комментариев
17. Реализован `createComment(w, r)`:
    - Извлекает `userID` из контекста
    - Парсит `id` (id поста)
    - Парсит JSON с `body`
    - Валидирует `body` (не пустой, не длиннее 2000)
    - Создаёт комментарий
    - **Если получен `ErrPostNotFound`** (через FK violation в Create) → 404
    - При успехе → 201 + JSON

### Регистрация роутов

18. `GET /posts/{id}/comments` — публичный
19. `POST /posts/{id}/comments` — через middleware

### Main

20. `commentRepo := storage.NewCommentRepository(db)`
21. Передаётся в `NewHandler`

### Поведение

22. Все сценарии из примера работают
23. Пустой список возвращает `[]`, не `null`
24. Удаление поста каскадно удаляет его комментарии
25. Удаление пользователя каскадно удаляет его комментарии
26. Эндпоинты модулей 1-4 продолжают работать

## Шаги

1. Создай миграцию `003_create_comments.sql` и примени
2. Создай `internal/domain/comment.go`
3. Добавь `isForeignKeyViolation` в `internal/storage/errors.go`
4. Создай `internal/storage/comment.go` с `CommentRepository`
5. Создай `internal/api/comments.go` с handlers
6. Обнови `Handler` — добавь поле `comments`
7. Обнови `NewHandler` — принимает `comments`
8. Обнови `RegisterRoutes` — добавь два эндпоинта
9. Обнови `main.go` — создай `commentRepo`
10. Запусти, прогон полного сценария

## Эксперименты

1. **Каскад при удалении пользователя:** создай пользователя, посты от него, комментарии к этим постам **И** комментарии этого пользователя на чужих постах. Удали пользователя через `psql`. Что осталось? Все его посты, все комментарии (свои и к своим постам) — удалены. Чужие посты — остались, но без комментариев от удалённого пользователя.

2. **Прямой INSERT с несуществующим post_id:**
   ```sql
   INSERT INTO comments (post_id, author_id, body) VALUES (9999, 1, 'test');
   ```
   Получишь FK violation. Это работа `REFERENCES posts(id)`.

3. **Прямой INSERT с пустым body:**
   ```sql
   INSERT INTO comments (post_id, author_id, body) VALUES (1, 1, '');
   ```
   CHECK violation. Это работа `CHECK (length(body) > 0)`.

4. **Создай комментарий к посту чужого пользователя:** регистрация Льва → пост Льва → токен Ани → POST `/posts/{lev_post_id}/comments` от Ани. Должно работать. Любой залогиненный может комментировать любой пост.

5. **EXPLAIN на запрос с фильтром по post_id:**
   ```sql
   EXPLAIN ANALYZE SELECT * FROM comments WHERE post_id = 1;
   ```
   Должен использовать `idx_comments_post_id`. Это критично для производительности — без индекса каждый запрос делал бы full scan.

6. **Заметь: комментарии в порядке `ASC`** — старые сначала. Сравни с постами (`DESC`). Это разные UX-решения: посты как лента (новое сверху), комментарии как дискуссия (по порядку).

7. **Попробуй прислать комментарий длиннее 2000 символов:**
   ```bash
   BODY=$(python3 -c "print('x' * 3000)")
   curl -X POST http://localhost:8080/posts/1/comments -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"body\":\"$BODY\"}"
   ```
   400 от Go-валидации. Если бы Go пропустил — БД отклонила бы из-за CHECK.

8. **Удалили пост с комментариями:** проверь через `psql` до и после `DELETE /posts/{id}`. Комментарии должны исчезнуть автоматически.

9. **(Опционально) Добавь pagination:** `GET /posts/{id}/comments?limit=10&offset=0`. Это `LIMIT $2 OFFSET $3` в SELECT. Парсинг query через `r.URL.Query().Get("limit")`. Полезное упражнение.

## Что показать Claude

Покажи Claude свой `createComment` и спроси: «правильно ли я обрабатываю случай несуществующего поста? Что лучше — проверять существование `GetByID` перед `Create`, или полагаться на FK violation?». Ответ:

- **Полагаться на FK** (наш подход) — один запрос вместо двух, атомарно. **Минус**: ошибка приходит как технический FK violation, нужно её парсить.
- **Проверять перед** — два запроса (`GetByID` + `Create`), есть **race condition**: пост может быть удалён между ними. **Минус**: больше кода, медленнее.

В нашем `createComment` мы используем FK violation. В `listComments` — `GetByID` (потому что нет INSERT, который мог бы упасть с FK).

Запиши себе на будущее: **полагаться на constraints БД — это норма**. БД даёт тебе **атомарные** проверки бесплатно, не пытайся их дублировать в коде. Дублирование — это когда ты делаешь `SELECT ... LIMIT 1` перед `INSERT` для проверки уникальности. Не делай так — используй `UNIQUE` constraint и лови ошибку.
