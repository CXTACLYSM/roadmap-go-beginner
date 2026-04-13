# Модуль 5. Комментарии и JOIN — чек-лист готовности

## Поведенческие маркеры

### Миграция

- [ ] Создан `migrations/003_create_comments.sql`
- [ ] **Два FK**: `post_id REFERENCES posts(id) ON DELETE CASCADE`, `author_id REFERENCES users(id) ON DELETE CASCADE`
- [ ] CHECK constraint на `body` (длина 1-2000)
- [ ] Два индекса: `idx_comments_post_id` и `idx_comments_author_id`
- [ ] Миграция применена

### Domain

- [ ] Тип `Comment` с полями `ID`, `PostID`, `AuthorID`, `Body`, `CreatedAt`
- [ ] Все поля имеют JSON-теги
- [ ] Тип `CommentWithAuthor` через embedded `Comment` + `Author User`

### Storage

- [ ] Реализована `isForeignKeyViolation(err) bool` (код `23503`)
- [ ] Создан `internal/storage/comment.go`
- [ ] Тип `CommentRepository` с `db *sql.DB`
- [ ] Конструктор `NewCommentRepository`
- [ ] Реализован `Create(ctx, postID, authorID, body) (*domain.Comment, error)`
- [ ] **`Create` ловит FK violation** и возвращает `ErrPostNotFound`
- [ ] Реализован `ListByPost(ctx, postID) ([]domain.CommentWithAuthor, error)`
- [ ] `ListByPost` использует **JOIN на users**
- [ ] `ListByPost` сортирует **`ORDER BY created_at ASC`**
- [ ] `ListByPost` возвращает `[]domain.CommentWithAuthor{}` для пустого случая
- [ ] `defer rows.Close()` присутствует
- [ ] Все методы используют `*Context`

### API

- [ ] Тип `Handler` имеет поле `comments *storage.CommentRepository`
- [ ] `NewHandler` принимает `comments`
- [ ] Создан `internal/api/comments.go`
- [ ] Реализован `listComments(w, r)`
- [ ] **`listComments` сначала проверяет существование поста** через `posts.GetByID`
- [ ] Если поста нет → 404
- [ ] Реализован `createComment(w, r)`
- [ ] `createComment` извлекает `userID` из контекста
- [ ] Парсит `id` (это id поста) через `parseInt64`
- [ ] Парсит JSON с `body`
- [ ] Валидирует body (не пустой, не длиннее 2000)
- [ ] При `ErrPostNotFound` (через FK violation) → 404
- [ ] При успехе → 201 + JSON

### Регистрация роутов

- [ ] `GET /posts/{id}/comments` — публичный
- [ ] `POST /posts/{id}/comments` — через middleware

### Main

- [ ] `commentRepo := storage.NewCommentRepository(db)`
- [ ] Передаётся в `NewHandler`

### Поведение

- [ ] Список комментариев пустой → `[]`, не `null`
- [ ] Создание комментария с токеном → 201
- [ ] Создание без токена → 401
- [ ] Создание к несуществующему посту → 404
- [ ] Список к несуществующему посту → 404
- [ ] Список к существующему пустому посту → 200 + `[]`
- [ ] Список с комментариями → 200 + JSON с авторами
- [ ] **Удаление поста** каскадно удаляет его комментарии
- [ ] **Удаление пользователя** каскадно удаляет его комментарии
- [ ] Все эндпоинты модулей 1-4 продолжают работать
- [ ] Код закоммичен

## Концептуальные маркеры

- [ ] Знаю, что у `comments` **два** foreign key
- [ ] Понимаю, как **CASCADE распространяется по нескольким уровням**
- [ ] Знаю, что такое **sub-resource паттерн** в URL
- [ ] Могу объяснить, **почему комментарии сортируются ASC**, а посты — DESC
- [ ] Знаю, что код PostgreSQL **`23503`** = foreign key violation
- [ ] Понимаю, **зачем** в `listComments` сначала проверять существование поста
- [ ] Знаю, что **полагаться на FK в БД** часто лучше, чем проверять руками
- [ ] Могу объяснить, что такое **«N+1 problem»** (хотя мы её не решаем)
- [ ] Понимаю, **зачем** index на каждую FK-колонку

## Маркеры экспериментов

- [ ] Проверил **каскад при удалении пользователя** (посты + комментарии исчезают)
- [ ] Прямой INSERT с несуществующим post_id → FK violation
- [ ] Прямой INSERT с пустым body → CHECK violation
- [ ] Создал комментарий к чужому посту, увидел, что разрешено
- [ ] Сделал EXPLAIN на SELECT с фильтром по post_id, увидел использование индекса
- [ ] Проверил, что `ORDER BY ASC` даёт старые комментарии сначала
- [ ] Прислал слишком длинный комментарий, увидел 400
- [ ] Удалил пост, проверил через `psql`, что комментарии исчезли

## Самопроверка

Не подсматривая, ответь:

1. Сколько FK у `comments` и куда они ссылаются?
2. Что произойдёт при удалении пользователя с 5 постами и 10 комментариями?
3. Какой код PostgreSQL означает FK violation?
4. Как структурирован URL для комментариев и почему?
5. Зачем `listComments` проверяет существование поста?
6. Зачем индекс на `post_id`?

## Подготовка к модулю 6

В модуле 6 — **финальный** модуль. Ты добавишь:

- **Тесты** для handlers через `httptest.NewRecorder` и table-driven approach
- **`testcontainers`** для интеграционных тестов с реальной PostgreSQL в Docker
- **Test fixtures** для подготовки данных
- **Graceful shutdown** через `http.Server.Shutdown(ctx)` и сигналы

После модуля 6 у тебя будет **полностью production-ready** сервис, который не стыдно показать на собеседовании. Это финал roadmap-а.
