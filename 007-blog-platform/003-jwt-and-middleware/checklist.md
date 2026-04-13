# Модуль 3. JWT и middleware — чек-лист готовности

## Поведенческие маркеры

### Зависимость

- [ ] Установлен `github.com/golang-jwt/jwt/v5`

### Auth

- [ ] К `auth.Service` добавлен `GenerateToken(userID int64) (string, error)`
- [ ] К `auth.Service` добавлен `ValidateToken(tokenStr) (int64, error)`
- [ ] Объявлена `ErrInvalidToken`
- [ ] Используется `jwt.SigningMethodHS256`
- [ ] Claims содержат `sub`, `exp` (через `time.Now().Add(24h).Unix()`), `iat`
- [ ] **Callback в `jwt.Parse` проверяет `t.Method.(*jwt.SigningMethodHMAC)`**
- [ ] `sub` извлекается через `claims["sub"].(float64)` и приводится к `int64`
- [ ] Любая ошибка валидации возвращает `ErrInvalidToken`

### Storage

- [ ] Добавлен `GetByID(ctx, id) (*domain.User, error)`
- [ ] Использует `QueryRowContext`
- [ ] При `sql.ErrNoRows` → `ErrUserNotFound`

### API — login

- [ ] Реализован handler `login`
- [ ] Парсит JSON с email и password
- [ ] Email нормализуется
- [ ] При `ErrUserNotFound` → 401 «invalid credentials»
- [ ] При `ErrInvalidPassword` → 401 **«invalid credentials»** (одинаковое сообщение!)
- [ ] При успехе → 200 + `{"token": "..."}`
- [ ] Эндпоинт `POST /login` зарегистрирован

### API — middleware

- [ ] Создан `internal/api/middleware.go`
- [ ] Объявлен **свой тип** `type contextKey string`
- [ ] Объявлен `userIDKey contextKey = "user_id"`
- [ ] Реализован `(h *Handler) authMiddleware(next http.Handler) http.Handler`
- [ ] Middleware проверяет наличие заголовка `Authorization`
- [ ] Middleware парсит формат `Bearer <token>` через `SplitN`
- [ ] Middleware валидирует токен через `h.auth.ValidateToken`
- [ ] Middleware кладёт `userID` в контекст через `context.WithValue`
- [ ] Middleware вызывает `next.ServeHTTP(w, r.WithContext(ctx))`
- [ ] Все ошибки middleware → 401 с осмысленным сообщением
- [ ] Реализован `userIDFromContext(ctx) (int64, bool)`

### API — /me

- [ ] Реализован handler `me`
- [ ] Извлекает userID через `userIDFromContext`
- [ ] Получает пользователя через `h.users.GetByID`
- [ ] Возвращает 200 + JSON
- [ ] При `ErrUserNotFound` → 404
- [ ] Эндпоинт `GET /me` зарегистрирован **через middleware** (`mux.Handle(...)`)

### Поведение

- [ ] Регистрация → логин → токен работает
- [ ] `/me` с валидным токеном → 200 + данные
- [ ] `/me` без токена → 401
- [ ] `/me` с битым токеном → 401
- [ ] `/me` с неправильным форматом → 401
- [ ] Логин с неправильным паролем → 401 «invalid credentials»
- [ ] Логин с несуществующим email → 401 **«invalid credentials»** (то же самое!)
- [ ] Все эндпоинты модулей 1-2 продолжают работать
- [ ] Код закоммичен

## Концептуальные маркеры

- [ ] Знаю, что такое JWT и из каких трёх частей состоит
- [ ] Понимаю, что JWT — **это не шифрование**, а подпись
- [ ] Знаю стандартные claims: `sub`, `exp`, `iat`
- [ ] Понимаю, **зачем** в callback проверять алгоритм подписи
- [ ] Знаю, что **JWT не отзываемый «из коробки»**
- [ ] Понимаю, что такое **user enumeration** и как от него защищаться
- [ ] Могу объяснить, что такое **middleware**
- [ ] Знаю стандартную сигнатуру middleware в Go: `func(next http.Handler) http.Handler`
- [ ] Понимаю, **зачем** свой тип для ключа в контексте (защита от коллизий)
- [ ] Могу объяснить, **что делает** `r.WithContext(ctx)`
- [ ] Знаю формат `Authorization: Bearer <token>`
- [ ] Понимаю разницу между `mux.HandleFunc` и `mux.Handle`

## Маркеры экспериментов

- [ ] **Открыл свой токен на jwt.io**, увидел читаемый payload
- [ ] Попробовал подменить payload без секрета, увидел отказ
- [ ] (Опционально) Подделал токен с правильным секретом, увидел, что сервер принимает
- [ ] (Опционально) Поставил короткий `tokenLifetime`, увидел истечение
- [ ] Проверил, что сообщения «нет пользователя» и «неправильный пароль» одинаковые
- [ ] Замерил время `Login` (большая часть — bcrypt)
- [ ] Использовал `jq` для извлечения токена и подстановки в следующий curl

## Самопроверка

Не подсматривая, ответь:

1. Из каких трёх частей состоит JWT?
2. Какой алгоритм мы используем и почему он симметричный?
3. Какие три claims минимально нужны?
4. Зачем callback в `jwt.Parse` проверяет тип метода?
5. Почему `sub` извлекается как `float64`?
6. Что должно возвращаться при «нет такого пользователя» в `/login`?
7. Как middleware передаёт `user_id` в handler?

## Подготовка к модулю 4

В модуле 4 ты добавишь **посты с владельцем**:

- Таблица `posts` с FK на `users` (`author_id`)
- ON DELETE CASCADE — что произойдёт с постами при удалении пользователя
- CRUD для постов: GET (публично), POST/PUT/DELETE (защищённо)
- **Проверка владения** в PUT/DELETE — только автор может править/удалять
- Возврат поста с информацией об авторе через JOIN

К концу модуля 4 у тебя будет настоящий блог: каждый зарегистрированный пользователь сможет писать свои посты, а другие — читать.
