# Модуль 1. Регистрация и токен — условие

## Задание

Зарегистрировать бота через @BotFather, получить токен, написать минимальную Go-программу, которая через `getMe` подтверждает, что токен работает и бот доступен.

## Что должно произойти

```bash
$ BOT_TOKEN="1234567890:ABCdefGhIJKlmNoPqRsTuVwXyZ" go run .
2026/04/13 12:00:00 Бот авторизован: id=1234567890 username=@lev_todo_bot name="Lev Todo Bot"
```

При неправильном токене:

```bash
$ BOT_TOKEN="неправильный_токен" go run .
2026/04/13 12:00:00 getMe: telegram error 401: Unauthorized
exit status 1
```

При отсутствии переменной:

```bash
$ go run .
2026/04/13 12:00:00 BOT_TOKEN не задан
exit status 1
```

## Требования

### Регистрация бота

1. Найти в Telegram **@BotFather**
2. Отправить `/newbot`
3. Ввести имя (любое, например, «Lev Todo Bot»)
4. Ввести username, оканчивающийся на `bot` (например, `lev_todo_bot`)
5. Получить токен в ответном сообщении
6. **Сохранить токен в безопасном месте** (НЕ в коде, НЕ в git)

### Программа

7. Создать папку `006-todo-bot/practice-setup/` с собственным `go.mod`
8. Создать `main.go`
9. Реализован тип `TelegramClient` с полями `token`, `client`, `apiURL`
10. Реализован конструктор `NewTelegramClient(token string) *TelegramClient`
11. `http.Client` создаётся с **таймаутом 30 секунд** (не `DefaultClient`)
12. Реализован метод `(c *TelegramClient) GetMe() (*User, error)`
13. Объявлен тип `User` с полями `ID`, `IsBot`, `FirstName`, `Username` (с JSON-тегами)
14. Объявлен внутренний тип `apiResponse` для парсинга ответа Telegram
15. Метод `GetMe`:
    - Делает HTTP GET по `https://api.telegram.org/bot<TOKEN>/getMe`
    - Читает тело через `io.ReadAll`
    - **Использует `defer resp.Body.Close()`**
    - Парсит в `apiResponse`
    - Проверяет `Ok` — если `false`, возвращает ошибку с `ErrorCode` и `Description`
    - Если `Ok` — парсит `Result` в `User` и возвращает
16. В `main`:
    - Читает `BOT_TOKEN` из `os.Getenv`
    - При отсутствии — `log.Fatal`
    - Создаёт клиент
    - Вызывает `GetMe`
    - При ошибке — `log.Fatalf`
    - При успехе — выводит `id`, `username`, `first_name`
17. **Токена в коде нет.** Только через переменную окружения.
18. **Файл с токеном не закоммичен в git.** Если используешь `.env` — он в `.gitignore`.

## Шаги

1. Открой Telegram, найди `@BotFather` (синяя галочка верификации)
2. Напиши `/newbot`, следуй инструкциям
3. Сохрани токен в `~/.todo_bot_token` (или другом локальном файле)
4. Создай папку проекта:
   ```bash
   mkdir -p 006-todo-bot/practice-setup
   cd 006-todo-bot/practice-setup
   go mod init bot-setup
   ```
5. Создай `main.go` с типами и функциями
6. Запусти:
   ```bash
   BOT_TOKEN="$(cat ~/.todo_bot_token)" go run .
   ```
   Или просто:
   ```bash
   BOT_TOKEN="1234567890:ABC..." go run .
   ```
7. Должно вывести информацию о боте

## Эксперименты

1. **Запусти без BOT_TOKEN** — увидь, как программа выходит с понятным сообщением.

2. **Запусти с неправильным токеном** (например, удали последний символ):
   ```bash
   BOT_TOKEN="123:wrong" go run .
   ```
   Что в ответе? `Telegram error 401: Unauthorized`. Это означает «токен не валидный».

3. **Открой Telegram Bot API URL в браузере:**
   ```
   https://api.telegram.org/bot<TOKEN>/getMe
   ```
   Замени `<TOKEN>` на свой. В браузере увидишь сырой JSON. Это **то же самое**, что делает твоя программа — просто HTTP GET.

4. **Сделай тот же запрос через `curl`:**
   ```bash
   curl "https://api.telegram.org/bot${BOT_TOKEN}/getMe"
   ```
   Сравни вывод с тем, что показывает твоя программа. Это полезно для понимания: бот = HTTP-клиент.

5. **Сделай запрос к несуществующему методу:**
   ```bash
   curl "https://api.telegram.org/bot${BOT_TOKEN}/nonexistent"
   ```
   Что в ответе? `404 Not Found` с понятным описанием. Telegram возвращает ошибки осмысленно.

6. **Найди своего бота в Telegram** по username, напиши ему `/start`. Что произошло? **Ничего**. Сообщение лежит на серверах Telegram и ждёт, когда твоя программа его заберёт. У тебя пока нет `getUpdates` — это будет в модуле 2.

7. **Установи описание бота** через @BotFather:
   - Напиши `/mybots`
   - Выбери своего бота
   - `Edit Bot` → `Edit Description`
   - Введи: «Бот для управления списком задач»
   - Открой профиль бота в Telegram — должно быть это описание

8. **Установи команды:**
   - `/mybots` → твой бот → `Edit Bot` → `Edit Commands`
   - Введи (по одной на строку):
     ```
     start - Начать
     help - Справка
     list - Показать задачи
     add - Добавить задачу
     done - Отметить выполненной
     delete - Удалить
     ```
   - Открой бота, начни вводить `/` — увидишь подсказки команд. Это и есть результат `setcommands`.

9. **Попробуй HTTP POST вместо GET:**
   ```bash
   curl -X POST "https://api.telegram.org/bot${BOT_TOKEN}/getMe"
   ```
   Telegram принимает оба. Это удобно — для разных методов разные привычки.

10. **Замерь время `GetMe`:**
    ```go
    start := time.Now()
    me, err := client.GetMe()
    log.Printf("getMe заняло %v", time.Since(start))
    ```
    Обычно 100-300ms (зависит от пинга до серверов Telegram). Это нормально для одного запроса. В `getUpdates` ты увидишь, что long polling устроен иначе.

## Что показать Claude

Покажи Claude свой код и спроси: «как мне организовать чтение токена из файла, а не из переменной окружения?». Это хороший вопрос для практики:

- **Простое решение** — `os.ReadFile(".token")`, добавить `.token` в `.gitignore`
- **Лучшее решение** — `.env` файл и библиотека вроде `godotenv`
- **Production-grade** — секрет-менеджер (vault, AWS Secrets Manager) или kubernetes secret

Для нашего проекта переменная окружения через `BOT_TOKEN=... go run .` достаточно. Но **знание про варианты** — обязательно.

Запиши себе на будущее: **никогда не делай токены/пароли литералами в коде**, даже временно «для теста». Один случайный коммит — и твой токен в публичном репозитории навсегда (история git помнит даже удалённые файлы).
