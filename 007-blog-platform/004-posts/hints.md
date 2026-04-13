# Модуль 4. Посты с владельцем — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какое ключевое слово SQL создаёт связь с другой таблицей?
2. Какое поведение `ON DELETE` выбираем для блога?
3. Зачем нужен индекс на FK-колонку?
4. Что такое **проверка владения**?
5. Какой статус возвращать при попытке редактировать чужой ресурс?

## Уровень 2: про FOREIGN KEY синтаксис

```sql
CREATE TABLE posts (
    id         BIGSERIAL PRIMARY KEY,
    author_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title      TEXT NOT NULL,
    body       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`REFERENCES users(id)` — FK. Можно расширить:
- `REFERENCES users(id) ON DELETE CASCADE` — каскадное удаление
- `REFERENCES users(id) ON DELETE RESTRICT` — запретить удаление
- `REFERENCES users(id) ON DELETE SET NULL` — обнулить

Можно также дать имя constraint-у:
```sql
CONSTRAINT fk_posts_author FOREIGN KEY (author_id) REFERENCES users(id) ON DELETE CASCADE
```

Это полезно для отладки (имя в сообщениях об ошибках).

## Уровень 3: про embedded struct

```go
type Post struct {
    ID    int64  `json:"id"`
    // ...
}

type PostWithAuthor struct {
    Post              // ← встроенный (embedded)
    Author User `json:"author"`
}
```

Embedded означает, что поля `Post` **автоматически становятся доступны** в `PostWithAuthor`:

```go
pa := PostWithAuthor{...}
pa.ID            // это pa.Post.ID, но можно обратиться напрямую
pa.Title         // pa.Post.Title
pa.Author.Email  // только Author не embedded — нужен префикс
```

В JSON-вывод поля `Post` попадают **на тот же уровень**, что `Author`:

```json
{
  "id": 1,           // от Post
  "author_id": 1,    // от Post
  "title": "...",    // от Post
  ...
  "author": {        // отдельное поле
    "id": 1,
    ...
  }
}
```

## Уровень 4: про JOIN в SELECT

```sql
SELECT
    p.id, p.author_id, p.title, p.body, p.created_at, p.updated_at,
    u.id, u.email, u.created_at
FROM posts p
JOIN users u ON u.id = p.author_id
WHERE p.id = $1
```

- **`FROM posts p`** — alias `p` для `posts`
- **`JOIN users u ON u.id = p.author_id`** — соединить с `users`, alias `u`, условие соединения
- **`SELECT p.id, ..., u.id, ...`** — колонки из обеих таблиц с префиксами

**Порядок колонок в SELECT** определяет порядок в `Scan`. Сначала колонки `p`, потом `u`:

```go
err := r.db.QueryRowContext(ctx, query, id).Scan(
    &pa.Post.ID, &pa.Post.AuthorID, &pa.Post.Title, &pa.Post.Body, &pa.Post.CreatedAt, &pa.Post.UpdatedAt,
    &pa.Author.ID, &pa.Author.Email, &pa.Author.CreatedAt,
)
```

Если перепутаешь порядок — runtime ошибка типов или странные данные.

## Уровень 5: про updated_at

```sql
UPDATE posts SET title = $1, body = $2, updated_at = now() WHERE id = $3
RETURNING ...
```

`updated_at = now()` обновляется **вручную** при каждом UPDATE. Альтернативно — через триггер на уровне БД (продвинутая тема).

## Уровень 6: про parseInt64

```go
func parseInt64(w http.ResponseWriter, r *http.Request, name string) (int64, bool) {
    s := r.PathValue(name)
    n, err := strconv.ParseInt(s, 10, 64)
    if err != nil {
        writeError(w, http.StatusBadRequest, "invalid id")
        return 0, false
    }
    return n, true
}
```

Похожий helper был в Проектах 4-5. Helper:
- Парсит path-параметр
- Возвращает `(int64, bool)`
- Сам пишет ошибку при неудаче
- Вызывающий просто `return` при `!ok`

Использование:
```go
id, ok := parseInt64(w, r, "id")
if !ok {
    return
}
// дальше работа с id
```

## Уровень 7: про шаблон handler с владением

```go
func (h *Handler) updatePost(w http.ResponseWriter, r *http.Request) {
    // 1. Кто я (из middleware)
    userID, ok := userIDFromContext(r.Context())
    if !ok {
        writeError(w, http.StatusInternalServerError, "user not in context")
        return
    }

    // 2. Что обновлять (из path)
    id, ok := parseInt64(w, r, "id")
    if !ok {
        return
    }

    // 3. Существует ли это
    existing, err := h.posts.GetByID(r.Context(), id)
    if err != nil {
        if errors.Is(err, storage.ErrPostNotFound) {
            writeError(w, http.StatusNotFound, "post not found")
            return
        }
        log.Printf("get post: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    // 4. Моё ли это
    if existing.Post.AuthorID != userID {
        writeError(w, http.StatusForbidden, "forbidden: not the author")
        return
    }

    // 5. Парсим вход
    var input struct { Title, Body string }
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "invalid json")
        return
    }

    // 6. Валидация
    input.Title = strings.TrimSpace(input.Title)
    input.Body = strings.TrimSpace(input.Body)
    if input.Title == "" || input.Body == "" {
        writeError(w, http.StatusBadRequest, "title and body are required")
        return
    }

    // 7. Делаем
    updated, err := h.posts.Update(r.Context(), id, input.Title, input.Body)
    if err != nil {
        log.Printf("update post: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    // 8. Отвечаем
    writeJSON(w, http.StatusOK, updated)
}
```

Структура: 8 шагов. Если какой-то пропускаешь — у тебя проблема.

## Уровень 8: про обновление NewHandler

```go
type Handler struct {
    users *storage.UserRepository
    posts *storage.PostRepository  // ← новое
    auth  *auth.Service
}

func NewHandler(
    users *storage.UserRepository,
    posts *storage.PostRepository,
    auth *auth.Service,
) *Handler {
    return &Handler{
        users: users,
        posts: posts,
        auth:  auth,
    }
}
```

И в `main.go`:

```go
userRepo := storage.NewUserRepository(db)
postRepo := storage.NewPostRepository(db)  // ← новое
authService := auth.NewService(cfg.JWTSecret)
handler := api.NewHandler(userRepo, postRepo, authService)
```

## Уровень 9: про существующие посты при удалении пользователя

Если у пользователя есть посты и ты пытаешься его удалить **через `psql`**:

```sql
DELETE FROM users WHERE id = 1;
```

С `ON DELETE CASCADE` — сработает, посты удалятся.
С `ON DELETE RESTRICT` — упадёт: `update or delete on table "users" violates foreign key constraint`.

Это и есть выбор поведения. Для блога CASCADE удобен, для финансовых данных — RESTRICT.

## Уровень 10: про идиоматичный List

```go
func (r *PostRepository) List(ctx context.Context) ([]domain.PostWithAuthor, error) {
    rows, err := r.db.QueryContext(ctx, `
        SELECT
            p.id, p.author_id, p.title, p.body, p.created_at, p.updated_at,
            u.id, u.email, u.created_at
        FROM posts p
        JOIN users u ON u.id = p.author_id
        ORDER BY p.created_at DESC
    `)
    if err != nil {
        return nil, fmt.Errorf("query posts: %w", err)
    }
    defer rows.Close()

    posts := []domain.PostWithAuthor{}  // ← пустой, не nil
    for rows.Next() {
        var pa domain.PostWithAuthor
        if err := rows.Scan(
            &pa.Post.ID, &pa.Post.AuthorID, &pa.Post.Title, &pa.Post.Body, &pa.Post.CreatedAt, &pa.Post.UpdatedAt,
            &pa.Author.ID, &pa.Author.Email, &pa.Author.CreatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan post: %w", err)
        }
        posts = append(posts, pa)
    }
    return posts, rows.Err()
}
```

Заметь:
- **`defer rows.Close()`** сразу после `Query`
- **`posts := []domain.PostWithAuthor{}`** — пустой слайс, чтобы JSON был `[]`, а не `null`
- **`return posts, rows.Err()`** в конце — финальная проверка

## Если совсем тупик

Скажи Claude конкретно: «модуль 4 проекта 7, [конкретная проблема]». Самые частые ситуации:

- **«violates foreign key constraint»** → пытаешься сохранить пост с несуществующим author_id
- **«post.AuthorID == 0»** → не передаёшь userID в Create или забыл взять из контекста
- **«scan error: column count mismatch»** → порядок колонок в SELECT не совпадает с порядком в Scan
- **«user can edit any post»** → забыл проверку владения в `updatePost` или `deletePost`
- **«List возвращает null вместо []»** → не инициализировал слайс через `[]Type{}`
