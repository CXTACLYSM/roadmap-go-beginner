# Модуль 5. Комментарии и JOIN — теория

## Контекст

В модуле 4 у тебя есть посты с одним FK (на пользователей). В этом модуле — комментарии с **двумя FK**: на пост и на пользователя. Это первая «настоящая» **многотабличная схема**: три связанные сущности, JOIN с двумя таблицами, sub-resource паттерн в URL.

После этого модуля у тебя будет «полноценный блог»: пользователи, посты, комментарии. Это та схема, которую ты увидишь в любом обучающем материале по веб-разработке — потому что она хорошо иллюстрирует **связи между сущностями**.

## Ядро 1: схема таблицы comments

```sql
-- migrations/003_create_comments.sql
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

Несколько важных моментов:

1. **Два foreign key**:
   - `post_id REFERENCES posts(id) ON DELETE CASCADE` — комментарий принадлежит посту
   - `author_id REFERENCES users(id) ON DELETE CASCADE` — комментарий написан пользователем
2. **Оба с CASCADE**:
   - Удалили пост → его комментарии удалились
   - Удалили пользователя → его комментарии удалились (а также его посты, и комментарии к этим постам)
3. **Два индекса** — по одному на каждый FK
4. **CHECK constraint** на body — не пустой, не длиннее 2000 символов
5. **Нет `updated_at`** — для простоты комментарии не редактируются

## Ядро 2: каскадное удаление по нескольким уровням

Что произойдёт, если удалить пользователя?

```
DELETE FROM users WHERE id = 1;
```

PostgreSQL автоматически:
1. Удалит все **посты** этого пользователя (FK `posts.author_id ON DELETE CASCADE`)
2. Для каждого удалённого поста — удалит все его **комментарии** (FK `comments.post_id ON DELETE CASCADE`)
3. Удалит все **комментарии** этого пользователя на чужих постах (FK `comments.author_id ON DELETE CASCADE`)

Один `DELETE` → каскад из трёх. Это **атомарно** и быстро. Без CASCADE тебе пришлось бы делать всё вручную и в правильном порядке (сначала комментарии, потом посты, потом пользователя).

## Ядро 3: тип Comment

```go
// internal/domain/comment.go
package domain

import "time"

type Comment struct {
    ID        int64     `json:"id"`
    PostID    int64     `json:"post_id"`
    AuthorID  int64     `json:"author_id"`
    Body      string    `json:"body"`
    CreatedAt time.Time `json:"created_at"`
}

type CommentWithAuthor struct {
    Comment
    Author User `json:"author"`
}
```

То же, что для постов: базовый тип + версия с автором для удобства клиента.

## Ядро 4: CommentRepository

```go
// internal/storage/comment.go
package storage

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "blog/internal/domain"
)

var ErrCommentNotFound = errors.New("comment not found")

type CommentRepository struct {
    db *sql.DB
}

func NewCommentRepository(db *sql.DB) *CommentRepository {
    return &CommentRepository{db: db}
}

func (r *CommentRepository) Create(ctx context.Context, postID, authorID int64, body string) (*domain.Comment, error) {
    var c domain.Comment
    err := r.db.QueryRowContext(ctx, `
        INSERT INTO comments (post_id, author_id, body)
        VALUES ($1, $2, $3)
        RETURNING id, post_id, author_id, body, created_at
    `, postID, authorID, body).Scan(&c.ID, &c.PostID, &c.AuthorID, &c.Body, &c.CreatedAt)
    if err != nil {
        if isForeignKeyViolation(err) {
            return nil, ErrPostNotFound  // post не существует
        }
        return nil, fmt.Errorf("insert comment: %w", err)
    }
    return &c, nil
}

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

Несколько важных моментов:

1. **`isForeignKeyViolation`** — функция, аналогичная `isUniqueViolation`. Ловит код PostgreSQL `23503` (foreign key violation). Если попытались создать комментарий к несуществующему посту — вернётся `ErrPostNotFound`.

2. **`ListByPost` фильтрует по `post_id`** — это та же multi-tenant логика, что в Проекте 6: каждый комментарий принадлежит одному посту.

3. **`ORDER BY c.created_at ASC`** — комментарии **сначала старые**, потом новые. Это естественный порядок чтения дискуссии (в отличие от постов, где `DESC`).

4. **Нет методов `Update` и `Delete`** — для простоты комментарии не редактируются и не удаляются (кроме каскадного удаления вместе с постом или пользователем). Можно добавить как упражнение.

## Ядро 5: isForeignKeyViolation

Аналогично `isUniqueViolation` из модуля 2:

```go
// internal/storage/errors.go
import (
    "errors"
    "github.com/jackc/pgx/v5/pgconn"
)

func isUniqueViolation(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr.Code == "23505"  // unique_violation
    }
    return false
}

func isForeignKeyViolation(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr.Code == "23503"  // foreign_key_violation
    }
    return false
}
```

Коды PostgreSQL:
- `23505` — `unique_violation`
- `23503` — `foreign_key_violation`
- `23502` — `not_null_violation`
- `23514` — `check_violation`

Это **класс 23** — Integrity Constraint Violation. Полезно знать для отладки.

## Ядро 6: sub-resource паттерн в URL

Когда комментарии — это **подресурс** поста, URL отражает иерархию:

```
GET    /posts/{id}/comments       ← список комментариев к посту
POST   /posts/{id}/comments       ← добавить комментарий к посту
```

Это **REST-конвенция**: «комментарии не существуют сами по себе, они **принадлежат** посту».

Альтернатива — отдельный resource:

```
GET  /comments?post_id=1
POST /comments  (с post_id в теле)
```

Это тоже валидно, но менее «REST-fully». **Sub-resource** более выразителен.

## Ядро 7: handlers для комментариев

```go
// internal/api/comments.go
package api

import (
    "encoding/json"
    "errors"
    "log"
    "net/http"
    "strings"

    "blog/internal/storage"
)

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

Несколько важных моментов:

1. **`listComments` сначала проверяет, что пост существует** — даже если у поста 0 комментариев, мы хотим различить «пост есть, комментариев нет» (200 + пустой массив) и «такого поста нет» (404).

2. **`createComment` использует FK violation** для определения «нет такого поста» — это работает в одном запросе, без отдельного `GetByID`.

3. **`parseInt64(w, r, "id")`** — используем тот же параметр `id` (это ID **поста** из URL `/posts/{id}/comments`).

4. **Никакой проверки владения** для комментариев — любой залогиненный пользователь может комментировать любой пост. В реальном блоге могут быть проверки (заблокированные пользователи, премодерация), но для нас этого достаточно.

## Ядро 8: путь `/posts/{id}/comments` в Go 1.22+

Path patterns Go 1.22+ умеют path parameters в любом месте URL:

```go
mux.HandleFunc("GET /posts/{id}/comments", h.listComments)
mux.Handle("POST /posts/{id}/comments", h.authMiddleware(http.HandlerFunc(h.createComment)))
```

`r.PathValue("id")` достаёт значение `{id}` из URL — это id **поста** (по пути).

⚠️ **Тонкость**: имя `{id}` здесь — это id поста, **не** id комментария. Если бы у нас был эндпоинт типа `DELETE /posts/{post_id}/comments/{comment_id}`, нужно было бы два разных имени. У нас этого нет, но в реальных API так часто.

## Ядро 9: финальная карта эндпоинтов

После модуля 5 у твоего API будет:

```
# Auth
POST   /register
POST   /login
GET    /me                           (auth)

# Posts
GET    /posts
GET    /posts/{id}
POST   /posts                        (auth)
PUT    /posts/{id}                   (auth, ownership)
DELETE /posts/{id}                   (auth, ownership)

# Comments
GET    /posts/{id}/comments
POST   /posts/{id}/comments          (auth)
```

9 эндпоинтов. Это **достаточная** API для реального блога. С ним можно интегрировать любой клиент.

## Ядро 10: что НЕ делаем (но можно как упражнение)

- **Update и Delete для комментариев** — не реализуем, но логика идентична постам
- **Замена 1+N запросов на батч** — для каждого поста в `List` мы могли бы загружать комментарии. Это было бы 1 запрос за посты + N за комментарии (классическая «N+1 problem»). Мы её не решаем.
- **Pagination** — `LIMIT` и `OFFSET` в SELECT. Стандартное добавление.
- **Search** — full-text search PostgreSQL. Отдельная тема.
- **Уведомления автору при новом комментарии** — это уже event-driven архитектура, отдельная история.

Все эти темы — естественные **следующие шаги**.

## Финальная регистрация роутов

```go
func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    // Auth
    mux.HandleFunc("GET /hello", h.hello)
    mux.HandleFunc("POST /register", h.register)
    mux.HandleFunc("POST /login", h.login)
    mux.Handle("GET /me", h.authMiddleware(http.HandlerFunc(h.me)))

    // Posts
    mux.HandleFunc("GET /posts", h.listPosts)
    mux.HandleFunc("GET /posts/{id}", h.getPost)
    mux.Handle("POST /posts", h.authMiddleware(http.HandlerFunc(h.createPost)))
    mux.Handle("PUT /posts/{id}", h.authMiddleware(http.HandlerFunc(h.updatePost)))
    mux.Handle("DELETE /posts/{id}", h.authMiddleware(http.HandlerFunc(h.deletePost)))

    // Comments
    mux.HandleFunc("GET /posts/{id}/comments", h.listComments)
    mux.Handle("POST /posts/{id}/comments", h.authMiddleware(http.HandlerFunc(h.createComment)))
}
```

## Вопросы на понимание

1. Сколько foreign key у таблицы `comments`?
2. Что произойдёт при удалении пользователя, у которого 5 постов и 10 комментариев?
3. Что такое **sub-resource** паттерн в URL?
4. Почему в `comments` мы сортируем `ORDER BY created_at ASC`, а не `DESC`?
5. Какой код PostgreSQL означает foreign key violation?
6. Зачем `listComments` сначала проверяет существование поста?
7. Может ли пользователь комментировать чужой пост?
8. Что такое «N+1 problem» и почему мы её не решаем в нашем `List`?
9. Какие два индекса есть на таблице `comments` и зачем каждый?
10. Можно ли использовать одно имя `{id}` для разных значений в разных URL?
