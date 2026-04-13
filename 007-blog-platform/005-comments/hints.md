# Модуль 5. Комментарии и JOIN — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Сколько foreign key у таблицы `comments`?
2. Какой код PostgreSQL означает FK violation?
3. Как назвать `id` в URL `/posts/{id}/comments` (это id поста, а не комментария)?
4. Какой порядок сортировки для комментариев — ASC или DESC?
5. Как обработать «нет такого поста» при создании комментария?

## Уровень 2: про две foreign keys

```sql
post_id    BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
author_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
```

Две связи, оба с CASCADE. Это значит:
- Удалили пост → удалились его комментарии
- Удалили пользователя → удалились **все** его комментарии (на любых постах)

И индексы на оба:

```sql
CREATE INDEX idx_comments_post_id ON comments (post_id);
CREATE INDEX idx_comments_author_id ON comments (author_id);
```

## Уровень 3: про isForeignKeyViolation

```go
// internal/storage/errors.go
import (
    "errors"
    "github.com/jackc/pgx/v5/pgconn"
)

func isForeignKeyViolation(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr.Code == "23503"
    }
    return false
}
```

Аналогично `isUniqueViolation`, только код `23503`. Можешь положить в тот же файл.

## Уровень 4: про Create с обработкой FK

```go
func (r *CommentRepository) Create(ctx context.Context, postID, authorID int64, body string) (*domain.Comment, error) {
    var c domain.Comment
    err := r.db.QueryRowContext(ctx, `
        INSERT INTO comments (post_id, author_id, body)
        VALUES ($1, $2, $3)
        RETURNING id, post_id, author_id, body, created_at
    `, postID, authorID, body).Scan(&c.ID, &c.PostID, &c.AuthorID, &c.Body, &c.CreatedAt)
    if err != nil {
        if isForeignKeyViolation(err) {
            return nil, ErrPostNotFound
        }
        return nil, fmt.Errorf("insert comment: %w", err)
    }
    return &c, nil
}
```

⚠️ **Тонкость**: если упадёт FK на `author_id` (например, пользователь удалён между логином и созданием комментария — крайний случай), мы тоже вернём `ErrPostNotFound`, что неточно. Для production можно различать через `pgErr.ConstraintName`, но для нашего проекта упрощение допустимо.

## Уровень 5: про ListByPost с JOIN

```go
func (r *CommentRepository) ListByPost(ctx context.Context, postID int64) ([]domain.CommentWithAuthor, error) {
    rows, err := r.db.QueryContext(ctx, `
        SELECT
            c.id, c.post_id, c.author_id, c.body, c.created_at,
            u.id, u.email, u.created_at
        FROM comments c
        JOIN users u ON u.id = c.author_id
        WHERE c.post_id = $1
        ORDER BY c.created_at ASC
    `, postID)
    if err != nil {
        return nil, fmt.Errorf("query comments: %w", err)
    }
    defer rows.Close()

    comments := []domain.CommentWithAuthor{}
    for rows.Next() {
        var ca domain.CommentWithAuthor
        if err := rows.Scan(
            &ca.Comment.ID, &ca.Comment.PostID, &ca.Comment.AuthorID, &ca.Comment.Body, &ca.Comment.CreatedAt,
            &ca.Author.ID, &ca.Author.Email, &ca.Author.CreatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan comment: %w", err)
        }
        comments = append(comments, ca)
    }
    return comments, rows.Err()
}
```

Структура та же, что в `posts.List` модуля 4:
- JOIN с users
- WHERE по post_id
- ORDER BY created_at ASC (для комментариев — старые сначала)
- defer rows.Close()
- []domain.CommentWithAuthor{} вместо nil
- rows.Err() в конце

## Уровень 6: про listComments handler

```go
func (h *Handler) listComments(w http.ResponseWriter, r *http.Request) {
    postID, ok := parseInt64(w, r, "id")
    if !ok {
        return
    }

    // Проверим, что пост существует
    if _, err := h.posts.GetByID(r.Context(), postID); err != nil {
        if errors.Is(err, storage.ErrPostNotFound) {
            writeError(w, http.StatusNotFound, "post not found")
            return
        }
        log.Printf("get post: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    comments, err := h.comments.ListByPost(r.Context(), postID)
    if err != nil {
        log.Printf("list comments: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }
    writeJSON(w, http.StatusOK, comments)
}
```

Зачем `GetByID`? Потому что без него:

- Запрос к `/posts/9999/comments` вернул бы пустой массив `[]`
- Это вводит в заблуждение: «то ли пост есть и комментариев нет, то ли поста нет»

С проверкой:
- Пост есть, комментариев нет → 200 + `[]`
- Поста нет → 404 + ошибка

Это **более точное** API.

## Уровень 7: про createComment handler

```go
func (h *Handler) createComment(w http.ResponseWriter, r *http.Request) {
    userID, ok := userIDFromContext(r.Context())
    if !ok {
        writeError(w, http.StatusInternalServerError, "user not in context")
        return
    }

    postID, ok := parseInt64(w, r, "id")
    if !ok {
        return
    }

    var input struct {
        Body string `json:"body"`
    }
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "invalid json")
        return
    }
    input.Body = strings.TrimSpace(input.Body)
    if input.Body == "" {
        writeError(w, http.StatusBadRequest, "body is required")
        return
    }
    if len(input.Body) > 2000 {
        writeError(w, http.StatusBadRequest, "body too long (max 2000)")
        return
    }

    comment, err := h.comments.Create(r.Context(), postID, userID, input.Body)
    if err != nil {
        if errors.Is(err, storage.ErrPostNotFound) {
            writeError(w, http.StatusNotFound, "post not found")
            return
        }
        log.Printf("create comment: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    writeJSON(w, http.StatusCreated, comment)
}
```

Структура:
1. Кто я (из контекста)
2. К какому посту (из path)
3. Что писать (из тела)
4. Валидация
5. Создание
6. Обработка ошибок (FK violation → 404)
7. Ответ

## Уровень 8: про обновление Handler и main

```go
type Handler struct {
    users    *storage.UserRepository
    posts    *storage.PostRepository
    comments *storage.CommentRepository  // ← новое
    auth     *auth.Service
}

func NewHandler(
    users *storage.UserRepository,
    posts *storage.PostRepository,
    comments *storage.CommentRepository,
    auth *auth.Service,
) *Handler {
    return &Handler{
        users:    users,
        posts:    posts,
        comments: comments,
        auth:     auth,
    }
}
```

И в main:

```go
userRepo := storage.NewUserRepository(db)
postRepo := storage.NewPostRepository(db)
commentRepo := storage.NewCommentRepository(db)
authService := auth.NewService(cfg.JWTSecret)
handler := api.NewHandler(userRepo, postRepo, commentRepo, authService)
```

## Уровень 9: про path параметр в sub-resource

```go
mux.HandleFunc("GET /posts/{id}/comments", h.listComments)
mux.Handle("POST /posts/{id}/comments", h.authMiddleware(http.HandlerFunc(h.createComment)))
```

`{id}` — это **id поста**. Достаём через `r.PathValue("id")`. Если бы у нас был эндпоинт типа `DELETE /posts/{post_id}/comments/{comment_id}`, нужны были бы разные имена:

```go
mux.HandleFunc("DELETE /posts/{post_id}/comments/{comment_id}", h.deleteComment)
```

```go
postID := r.PathValue("post_id")
commentID := r.PathValue("comment_id")
```

В нашем модуле этого нет.

## Если совсем тупик

Скажи Claude конкретно: «модуль 5 проекта 7, [конкретная проблема]». Самые частые ситуации:

- **«violates foreign key constraint»** → пытаешься создать комментарий к несуществующему посту. Это как раз то, что мы должны ловить через `isForeignKeyViolation`.
- **«createComment всегда 404»** → может быть, путаешь имена параметров; проверь, что в URL `{id}` и в `r.PathValue("id")` совпадают
- **«list возвращает null»** → не инициализировал слайс через `[]Type{}`
- **«scan error: column count mismatch»** → JOIN возвращает 8 колонок (5+3), Scan ожидает другое число
