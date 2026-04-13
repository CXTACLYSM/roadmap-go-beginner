# Модуль 4. Посты с владельцем — чек-лист готовности

## Поведенческие маркеры

### Миграция

- [ ] Создан `migrations/002_create_posts.sql`
- [ ] FK `author_id REFERENCES users(id) ON DELETE CASCADE`
- [ ] CHECK constraint на `title` (длина 1-200)
- [ ] CHECK constraint на `body` (не пустой)
- [ ] Поле `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [ ] Индекс `idx_posts_author_id`
- [ ] Индекс `idx_posts_created_at` с `DESC`
- [ ] Миграция применена

### Domain

- [ ] Тип `Post` с полями `ID`, `AuthorID`, `Title`, `Body`, `CreatedAt`, `UpdatedAt`
- [ ] Все поля имеют JSON-теги
- [ ] Тип `PostWithAuthor` использует **embedded** `Post` + поле `Author User`

### Storage

- [ ] Создан `internal/storage/post.go`
- [ ] Тип `PostRepository` с полем `db *sql.DB`
- [ ] Конструктор `NewPostRepository`
- [ ] Объявлена `ErrPostNotFound`
- [ ] Реализован `Create(ctx, authorID, title, body) (*domain.Post, error)`
- [ ] Реализован `GetByID(ctx, id) (*domain.PostWithAuthor, error)` **с JOIN**
- [ ] Реализован `List(ctx) ([]domain.PostWithAuthor, error)` **с JOIN и ORDER BY DESC**
- [ ] Реализован `Update(ctx, id, title, body) (*domain.Post, error)` с `updated_at = now()`
- [ ] Реализован `Delete(ctx, id) error`
- [ ] Все методы используют `*Context` версии
- [ ] `GetByID`/`Update` возвращают `ErrPostNotFound` при `sql.ErrNoRows`
- [ ] `Delete` использует `RowsAffected`
- [ ] `List` возвращает `[]domain.PostWithAuthor{}` (не `nil`)
- [ ] `defer rows.Close()` в `List`

### API

- [ ] Тип `Handler` имеет поле `posts *storage.PostRepository`
- [ ] `NewHandler` принимает `posts`
- [ ] Создан `internal/api/posts.go`
- [ ] Реализованы 5 handlers: `listPosts`, `getPost`, `createPost`, `updatePost`, `deletePost`
- [ ] Helper `parseInt64(w, r, name) (int64, bool)` реализован

### Проверка владения

- [ ] `updatePost` сначала вызывает `GetByID`
- [ ] `updatePost` проверяет `existing.Post.AuthorID == userID`
- [ ] При несовпадении → **403 Forbidden** с сообщением `"forbidden: not the author"`
- [ ] То же самое в `deletePost`

### Регистрация роутов

- [ ] `GET /posts` — публичный
- [ ] `GET /posts/{id}` — публичный
- [ ] `POST /posts` — через middleware
- [ ] `PUT /posts/{id}` — через middleware
- [ ] `DELETE /posts/{id}` — через middleware

### Main

- [ ] `postRepo := storage.NewPostRepository(db)`
- [ ] Передаётся в `NewHandler`

### Поведение

- [ ] Создание поста с токеном работает (201)
- [ ] Создание без токена → 401
- [ ] Список постов публичен и содержит автора
- [ ] Один пост публичен и содержит автора
- [ ] Обновление своего поста работает (200)
- [ ] Обновление чужого поста → 403
- [ ] Удаление своего поста → 204
- [ ] Удаление чужого поста → 403
- [ ] Удаление несуществующего → 404
- [ ] Невалидный JSON → 400
- [ ] Пустой title или body → 400
- [ ] Слишком длинный title → 400
- [ ] Все эндпоинты модулей 1-3 продолжают работать
- [ ] Код закоммичен

## Концептуальные маркеры

- [ ] Знаю, что такое **foreign key** и какие гарантии он даёт
- [ ] Перечисляю несколько `ON DELETE` поведений и понимаю разницу
- [ ] Знаю, что **PostgreSQL не создаёт индекс** на FK автоматически
- [ ] Понимаю **разницу между authentication и authorization**
- [ ] Могу объяснить, что такое **проверка владения**
- [ ] Знаю разницу между **403 Forbidden** и **404 Not Found** для чужих ресурсов
- [ ] Могу объяснить, что такое **embedded struct** и как `PostWithAuthor` его использует
- [ ] Понимаю, **почему репозиторий не должен** проверять владение (это бизнес-логика)
- [ ] Знаю про **race condition** в двух-шаговом подходе и как её решить через `WHERE` в UPDATE
- [ ] Понимаю, **зачем** дублировать валидацию в Go и в БД (CHECK)
- [ ] Знаю, что такое **JOIN** и зачем он нужен для денормализованного ответа

## Маркеры экспериментов

- [ ] **Удалил пользователя через `psql`**, увидел, что его посты удалились (CASCADE)
- [ ] Попробовал INSERT с несуществующим author_id, увидел `violates foreign key constraint`
- [ ] Попробовал INSERT с пустым title через `psql`, увидел работу CHECK
- [ ] **Попробовал отредактировать чужой пост**, получил 403
- [ ] Создал несколько постов, проверил сортировку DESC по created_at
- [ ] Сделал EXPLAIN на SELECT с фильтром по author_id, увидел использование индекса
- [ ] (Опционально) Попробовал переписать `updatePost` через `WHERE id AND author_id`

## Самопроверка

Не подсматривая, ответь:

1. Какое поведение `ON DELETE` я выбрал для `posts.author_id` и почему?
2. Какие два индекса есть на таблице `posts`?
3. Что такое embedded struct в Go?
4. Какие 8 шагов в `updatePost`?
5. Какой статус возвращать при попытке редактировать чужой пост?
6. Зачем `List` возвращает `[]Type{}`, а не `nil`?

## Подготовка к модулю 5

В модуле 5 ты добавишь **комментарии**:

- Таблица `comments` с **двумя FK**: на `posts` и на `users`
- POST `/posts/{id}/comments` — sub-resource паттерн
- GET `/posts/{id}/comments` — список комментариев с авторами
- JOIN с `users` для каждого комментария
- Каскадное удаление: при удалении поста удаляются все его комментарии

К концу модуля 5 у тебя будет настоящий блог: посты, комментарии к ним, пользователи. Это **полная multi-table схема**, которую ты увидишь в реальных приложениях.
