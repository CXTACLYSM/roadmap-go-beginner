# Модуль 4. Посты с владельцем — теория

## Контекст

В модулях 2-3 у тебя есть пользователи и аутентификация. Теперь — **первая реальная сущность блога**: посты. Каждый пост принадлежит **одному** пользователю, и только этот пользователь может его редактировать и удалять.

В этом модуле:

- Создашь таблицу `posts` со связью на `users` через **foreign key**
- Узнаешь про `ON DELETE CASCADE` и `ON DELETE RESTRICT`
- Реализуешь CRUD для постов (5 эндпоинтов)
- Поймёшь разницу между **аутентификацией** (`authn`) и **авторизацией** (`authz`)
- Реализуешь **проверку владения** через сравнение `author_id` с `user_id` из контекста

После этого модуля у тебя будет «настоящий блог»: пользователи пишут свои посты, читать их может кто угодно, редактировать — только автор.

## Ядро 1: foreign key

**Foreign key** (внешний ключ) — это колонка в таблице, которая ссылается на колонку в **другой** таблице. Это создаёт **связь** между сущностями на уровне БД.

```sql
CREATE TABLE posts (
    id         BIGSERIAL PRIMARY KEY,
    author_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title      TEXT NOT NULL,
    body       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`REFERENCES users(id)` — это и есть foreign key. Он говорит: «значение колонки `author_id` **должно** соответствовать какому-то существующему `id` в таблице `users`».

Что это даёт:

1. **Целостность данных**: невозможно вставить пост с `author_id = 999`, если пользователя 999 нет. PostgreSQL отклонит INSERT.
2. **Гарантия при чтении**: если запись есть, ты **точно** знаешь, что есть и связанный пользователь
3. **Каскадные действия**: можно сказать БД, что делать при удалении ссылаемой записи

Без FK тебе пришлось бы **вручную** проверять существование пользователя перед каждым INSERT. С FK — БД делает это бесплатно и **атомарно**.

## Ядро 2: ON DELETE поведение

Когда удаляешь пользователя, у которого есть посты — что делать с постами? PostgreSQL предлагает несколько вариантов:

| Поведение | Что происходит |
|-----------|----------------|
| **`ON DELETE RESTRICT`** | Удалить пользователя **запрещено**, если есть посты. Пишешь — получаешь ошибку. |
| **`ON DELETE NO ACTION`** | Похоже на RESTRICT, но проверка отложенная (важно для транзакций). По умолчанию. |
| **`ON DELETE CASCADE`** | При удалении пользователя **автоматически удаляются** все его посты |
| **`ON DELETE SET NULL`** | При удалении `author_id` становится `NULL` (нужно `NULL` разрешён) |
| **`ON DELETE SET DEFAULT`** | Устанавливает default-значение |

Какой выбрать — **зависит от семантики**:

- **`CASCADE`** для блога: «удалили пользователя — удаляем его посты». Это естественно.
- **`RESTRICT`** для финансов: «нельзя удалить клиента, у которого есть транзакции». Транзакции — священны, их не удаляют просто так.
- **`SET NULL`** для редактуры: «удалили пользователя — посты остались, но без автора (anonymous)». Это нестандартно.

**Для нашего проекта — `CASCADE`.** Удалили пользователя — все его посты исчезают.

⚠️ В **продакшен-системах** часто **не удаляют** пользователей физически, а делают **soft delete** (флаг `deleted_at`). Тогда FK не срабатывает, и посты остаются. Это другой паттерн, о нём поговорим в Проектах после roadmap-а.

## Ядро 3: индексы для FK

PostgreSQL **не создаёт** индекс на FK автоматически (в отличие от MySQL). Это значит, что запросы вида:

```sql
SELECT * FROM posts WHERE author_id = 1;
```

— **по умолчанию делают full table scan** на posts. На большой таблице это медленно.

**Поэтому всегда создавай индекс на FK-колонку:**

```sql
CREATE INDEX idx_posts_author_id ON posts (author_id);
```

Это правило: **каждый foreign key → индекс**. Пропустить — частая ошибка, которая всплывает на масштабе.

## Ядро 4: authn vs authz

Эти два слова часто путают:

- **Authentication (authn)** — «**кто** ты». Проверка личности. Логин, токены, пароли.
- **Authorization (authz)** — «**что тебе можно**». Права. Можешь ли ты редактировать **этот конкретный** пост?

В нашем модуле:

- **Authn** уже работает (модуль 3): middleware проверяет токен и кладёт `user_id` в контекст
- **Authz** добавляется здесь: handler проверяет, что `post.AuthorID == userID` перед изменением

Самая простая форма authz — **проверка владения**: «ты автор этой штуки? Тогда можно». Это покрывает большинство случаев в небольших приложениях.

В больших системах authz сложнее: роли (admin, moderator), permissions, политики. Это тема для отдельного курса.

## Ядро 5: схема таблицы posts

```sql
-- migrations/002_create_posts.sql
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

Несколько важных моментов:

1. **`author_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE`** — foreign key с каскадом
2. **CHECK constraints** для title и body — БД сама запретит пустые посты или слишком длинные заголовки. Это **дополнительная** валидация поверх Go.
3. **`updated_at`** — будем обновлять при PUT
4. **`idx_posts_created_at`** для сортировки списка постов по времени (новые сначала)

## Ядро 6: тип Post

```go
// internal/domain/post.go
package domain

import "time"

type Post struct {
    ID        int64     `json:"id"`
    AuthorID  int64     `json:"author_id"`
    Title     string    `json:"title"`
    Body      string    `json:"body"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// PostWithAuthor — для отдачи поста с информацией об авторе
type PostWithAuthor struct {
    Post
    Author User `json:"author"`
}
```

`PostWithAuthor` — это **embedded struct**: `Post` встроен в новую структуру плюс добавлено поле `Author`. JSON-результат:

```json
{
  "id": 1,
  "author_id": 1,
  "title": "...",
  "body": "...",
  "created_at": "...",
  "updated_at": "...",
  "author": {
    "id": 1,
    "email": "lev@example.com",
    "created_at": "..."
  }
}
```

Это **денормализованный** ответ — клиенту удобнее не делать второй запрос.

## Ядро 7: PostRepository

```go
// internal/storage/post.go
package storage

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "blog/internal/domain"
)

var ErrPostNotFound = errors.New("post not found")

type PostRepository struct {
    db *sql.DB
}

func NewPostRepository(db *sql.DB) *PostRepository {
    return &PostRepository{db: db}
}

func (r *PostRepository) Create(ctx context.Context, authorID int64, title, body string) (*domain.Post, error) {
    var p domain.Post
    err := r.db.QueryRowContext(ctx, `
        INSERT INTO posts (author_id, title, body)
        VALUES ($1, $2, $3)
        RETURNING id, author_id, title, body, created_at, updated_at
    `, authorID, title, body).Scan(&p.ID, &p.AuthorID, &p.Title, &p.Body, &p.CreatedAt, &p.UpdatedAt)
    if err != nil {
        return nil, fmt.Errorf("insert post: %w", err)
    }
    return &p, nil
}

func (r *PostRepository) GetByID(ctx context.Context, id int64) (*domain.PostWithAuthor, error) {
    var pa domain.PostWithAuthor
    err := r.db.QueryRowContext(ctx, `
        SELECT
            p.id, p.author_id, p.title, p.body, p.created_at, p.updated_at,
            u.id, u.email, u.created_at
        FROM posts p
        JOIN users u ON u.id = p.author_id
        WHERE p.id = $1
    `, id).Scan(
        &pa.Post.ID, &pa.Post.AuthorID, &pa.Post.Title, &pa.Post.Body, &pa.Post.CreatedAt, &pa.Post.UpdatedAt,
        &pa.Author.ID, &pa.Author.Email, &pa.Author.CreatedAt,
    )
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrPostNotFound
        }
        return nil, fmt.Errorf("get post: %w", err)
    }
    return &pa, nil
}

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

    posts := []domain.PostWithAuthor{}
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

func (r *PostRepository) Update(ctx context.Context, id int64, title, body string) (*domain.Post, error) {
    var p domain.Post
    err := r.db.QueryRowContext(ctx, `
        UPDATE posts SET title = $1, body = $2, updated_at = now()
        WHERE id = $3
        RETURNING id, author_id, title, body, created_at, updated_at
    `, title, body, id).Scan(&p.ID, &p.AuthorID, &p.Title, &p.Body, &p.CreatedAt, &p.UpdatedAt)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrPostNotFound
        }
        return nil, fmt.Errorf("update post: %w", err)
    }
    return &p, nil
}

func (r *PostRepository) Delete(ctx context.Context, id int64) error {
    result, err := r.db.ExecContext(ctx, "DELETE FROM posts WHERE id = $1", id)
    if err != nil {
        return fmt.Errorf("delete post: %w", err)
    }
    affected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("rows affected: %w", err)
    }
    if affected == 0 {
        return ErrPostNotFound
    }
    return nil
}
```

Несколько важных моментов:

1. **JOIN в `GetByID` и `List`** — загружаем пост **вместе с автором** одним запросом. Альтернатива (два запроса: пост + пользователь) работает, но менее эффективна.

2. **Порядок колонок в SELECT и Scan** — критичен. Columns `p.*, u.*`, scan порядок такой же.

3. **`ORDER BY p.created_at DESC`** — новые посты сначала. Стандартное поведение для блога.

4. **`Update` НЕ проверяет владение** — это будет в handler. Репозиторий — «тупой», только SQL. Бизнес-правила (кто может редактировать) — в слое выше.

## Ядро 8: handlers с проверкой владения

```go
// internal/api/posts.go
package api

import (
    "encoding/json"
    "errors"
    "log"
    "net/http"
    "strconv"
    "strings"

    "blog/internal/storage"
)

func (h *Handler) listPosts(w http.ResponseWriter, r *http.Request) {
    posts, err := h.posts.List(r.Context())
    if err != nil {
        log.Printf("list posts: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }
    writeJSON(w, http.StatusOK, posts)
}

func (h *Handler) getPost(w http.ResponseWriter, r *http.Request) {
    id, ok := parseInt64(w, r, "id")
    if !ok {
        return
    }
    post, err := h.posts.GetByID(r.Context(), id)
    if err != nil {
        if errors.Is(err, storage.ErrPostNotFound) {
            writeError(w, http.StatusNotFound, "post not found")
            return
        }
        log.Printf("get post: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }
    writeJSON(w, http.StatusOK, post)
}

func (h *Handler) createPost(w http.ResponseWriter, r *http.Request) {
    userID, ok := userIDFromContext(r.Context())
    if !ok {
        writeError(w, http.StatusInternalServerError, "user not in context")
        return
    }

    var input struct {
        Title string `json:"title"`
        Body  string `json:"body"`
    }
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "invalid json")
        return
    }

    input.Title = strings.TrimSpace(input.Title)
    input.Body = strings.TrimSpace(input.Body)
    if input.Title == "" || input.Body == "" {
        writeError(w, http.StatusBadRequest, "title and body are required")
        return
    }
    if len(input.Title) > 200 {
        writeError(w, http.StatusBadRequest, "title too long (max 200)")
        return
    }

    post, err := h.posts.Create(r.Context(), userID, input.Title, input.Body)
    if err != nil {
        log.Printf("create post: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }
    writeJSON(w, http.StatusCreated, post)
}

func (h *Handler) updatePost(w http.ResponseWriter, r *http.Request) {
    userID, ok := userIDFromContext(r.Context())
    if !ok {
        writeError(w, http.StatusInternalServerError, "user not in context")
        return
    }

    id, ok := parseInt64(w, r, "id")
    if !ok {
        return
    }

    // 1. Получаем существующий пост
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

    // 2. ПРОВЕРКА ВЛАДЕНИЯ
    if existing.Post.AuthorID != userID {
        writeError(w, http.StatusForbidden, "forbidden: not the author")
        return
    }

    // 3. Парсим и валидируем ввод
    var input struct {
        Title string `json:"title"`
        Body  string `json:"body"`
    }
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "invalid json")
        return
    }
    input.Title = strings.TrimSpace(input.Title)
    input.Body = strings.TrimSpace(input.Body)
    if input.Title == "" || input.Body == "" {
        writeError(w, http.StatusBadRequest, "title and body are required")
        return
    }

    // 4. Обновляем
    updated, err := h.posts.Update(r.Context(), id, input.Title, input.Body)
    if err != nil {
        log.Printf("update post: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }
    writeJSON(w, http.StatusOK, updated)
}

func (h *Handler) deletePost(w http.ResponseWriter, r *http.Request) {
    userID, ok := userIDFromContext(r.Context())
    if !ok {
        writeError(w, http.StatusInternalServerError, "user not in context")
        return
    }

    id, ok := parseInt64(w, r, "id")
    if !ok {
        return
    }

    // Получаем пост, чтобы проверить владельца
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

    if existing.Post.AuthorID != userID {
        writeError(w, http.StatusForbidden, "forbidden: not the author")
        return
    }

    if err := h.posts.Delete(r.Context(), id); err != nil {
        log.Printf("delete post: %v", err)
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }
    w.WriteHeader(http.StatusNoContent)
}

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

## Ядро 9: главный паттерн `update/delete` с владением

Заметь структуру `updatePost` и `deletePost`:

1. **Извлечь user_id** из контекста (от middleware)
2. **Извлечь post_id** из path
3. **Получить существующий пост** через `GetByID`
4. **Проверить, что `post.AuthorID == userID`** → если нет, 403 Forbidden
5. **Только тогда** выполнить операцию

Это **классический паттерн авторизации**. Применим почти везде, где есть «ресурсы с владельцем».

⚠️ **Тонкая race condition**: между «получить пост» и «обновить» теоретически кто-то может удалить пост или поменять владельца. В небольших проектах это не критично, в больших — решается через `WHERE author_id = $userID` прямо в UPDATE/DELETE:

```sql
UPDATE posts SET title = $1, body = $2 WHERE id = $3 AND author_id = $4 RETURNING ...
```

И если `RowsAffected == 0` → «либо нет, либо не твой» → 404 или 403.

**Для модуля 4 — двух-шаговый подход** (GetByID + проверка). Понять разницу через эксперимент.

## Ядро 10: 403 vs 404 для чужих ресурсов

Когда пользователь пытается удалить чужой пост — что возвращать?

- **403 Forbidden** — «я знаю, кто ты, но тебе нельзя». Прямой ответ.
- **404 Not Found** — «такого поста нет (для тебя)». Скрывает существование.

Оба валидны. **403 более информативен** (пользователь понимает, что пост есть, но не его). **404 более безопасен** (атакующий не узнаёт, какие ресурсы существуют у других).

**Для блога 403 нормально** — посты публичные, скрывать нечего. **Для приватных данных** (например, чужие банковские счета) — лучше 404.

В нашем проекте — **403**.

## Регистрация роутов

```go
func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    // Из модулей 1-3
    mux.HandleFunc("GET /hello", h.hello)
    mux.HandleFunc("POST /register", h.register)
    mux.HandleFunc("POST /login", h.login)
    mux.Handle("GET /me", h.authMiddleware(http.HandlerFunc(h.me)))

    // Посты — публичные
    mux.HandleFunc("GET /posts", h.listPosts)
    mux.HandleFunc("GET /posts/{id}", h.getPost)

    // Посты — защищённые
    mux.Handle("POST /posts", h.authMiddleware(http.HandlerFunc(h.createPost)))
    mux.Handle("PUT /posts/{id}", h.authMiddleware(http.HandlerFunc(h.updatePost)))
    mux.Handle("DELETE /posts/{id}", h.authMiddleware(http.HandlerFunc(h.deletePost)))
}
```

И обнови `Handler`:

```go
type Handler struct {
    users *storage.UserRepository
    posts *storage.PostRepository  // ← новое
    auth  *auth.Service
}

func NewHandler(users *storage.UserRepository, posts *storage.PostRepository, auth *auth.Service) *Handler {
    return &Handler{users: users, posts: posts, auth: auth}
}
```

И в `main.go`:

```go
postRepo := storage.NewPostRepository(db)
handler := api.NewHandler(userRepo, postRepo, authService)
```

## Вопросы на понимание

1. Что такое **foreign key** и какие гарантии он даёт?
2. Какие пять `ON DELETE` поведений? Какое выбираем для блога и почему?
3. Почему **PostgreSQL не создаёт индекс** на FK автоматически?
4. В чём разница между **authentication** и **authorization**?
5. Что такое **проверка владения**?
6. Какой статус возвращать при попытке редактировать чужой пост — 403 или 404?
7. Что такое **embedded struct** в Go и как `PostWithAuthor` его использует?
8. Почему `Update` в репозитории НЕ проверяет владение?
9. Какой паттерн используется в `updatePost` для проверки владения?
10. Какая race condition есть в двух-шаговом подходе и как её решить через `WHERE` в UPDATE?
