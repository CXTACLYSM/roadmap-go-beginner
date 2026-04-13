# Модуль 1. Регистрация и токен — теория

## Контекст

В Проектах 4-5 ты писал **сервер**: программа, которая слушает порт и принимает запросы. В этом проекте всё наоборот: твоя программа — **клиент**, который сам отправляет запросы к **чужому** API (Telegram).

Прежде чем начать писать код, нужно понять, как это вообще работает: что такое бот, как Telegram отличает «своего» бота от чужого, как программа получает доступ. Этот модуль — **исключительно про настройку**. Ты не напишешь логику бота. Ты:

- Зарегистрируешь бота через специального бота-помощника Telegram
- Получишь **токен** — секретный ключ, который идентифицирует твоего бота
- Напишешь **минимальную** Go-программу, которая делает один HTTP-запрос к Telegram API и проверяет, что токен работает

Это «hello world» бота. Никаких сообщений, никакого взаимодействия — только подтверждение, что инфраструктура настроена.

## Ядро 1: что такое Telegram-бот

С точки зрения Telegram, **бот — это специальный аккаунт**, у которого:

- Нет пароля и не нужно входить через приложение
- Все действия совершаются через **HTTP API**
- Аутентификация — через **токен** (длинная строка вроде `1234567890:ABCdefGhIJKlmNoPqRsTuVwXyZ`)
- Имя начинается с «бот»-стиля и оканчивается на `bot` (например, `MyTaskBot`)

Когда ты пишешь боту в Telegram, твоё сообщение попадает на серверы Telegram. Они его сохраняют и **ждут**, пока твоя программа спросит «есть для меня сообщения?». Программа спрашивает через HTTP, получает JSON со списком сообщений, обрабатывает их, и отправляет ответы — тоже через HTTP.

Никакой магии. Бот — это **HTTP-клиент** к серверам Telegram. Всё, что он умеет — это:

- Спрашивать «есть новые события?» (метод `getUpdates`)
- Отправлять сообщения (`sendMessage`)
- Отправлять файлы, делать опросы, инлайн-кнопки и так далее (всё через свои методы API)

## Ядро 2: что такое @BotFather

**@BotFather** — это бот, который **создаёт** других ботов. Это специальный сервис Telegram. Принцип:

1. Ты пишешь @BotFather команду `/newbot`
2. Он спрашивает имя для бота (отображаемое имя)
3. Он спрашивает username (должен оканчиваться на `bot`, например, `lev_todo_bot`)
4. Если username свободен — он генерирует **токен** и присылает тебе

С этого момента у тебя есть рабочий бот. Ты можешь его найти в Telegram (по username), но он не отвечает — потому что **никто не запросил `getUpdates`**, никакой программы за ним нет.

@BotFather также позволяет:

- Установить **описание** бота (видно в его профиле)
- Установить **команды** (выпадающий список «синих» команд при вводе `/`)
- Установить аватар
- Управлять **inline mode**, **группами** и другими настройками

Самые важные команды @BotFather:

- `/newbot` — создать нового бота
- `/mybots` — список твоих ботов
- `/setdescription` — установить описание
- `/setcommands` — установить список команд
- `/token` — получить токен существующего бота (если ты потерял свой)
- `/deletebot` — удалить бота

## Ядро 3: токен — это пароль

Токен бота — это **полный доступ** к нему. Любой, кто знает токен, может:

- Читать **все** сообщения, отправленные боту
- Отправлять сообщения **от имени** бота любому, кто с ним взаимодействовал
- Менять настройки
- Удалить бота через @BotFather

Поэтому **токен — это секрет**. Правила:

1. **Никогда не коммить токен в git.** Это как коммитить пароль от продакшен-БД. Если случайно закоммитил — **немедленно** перевыпусти токен через @BotFather (`/token` → `/revoke`).
2. **Не отправляй токен в чатах, документах, скриншотах.** Даже друзьям. Это доступ к боту.
3. **Храни в переменных окружения** в коде, а не как литерал.
4. **Для продакшена** — храни в секрет-менеджере (vault, AWS Secrets Manager, и так далее).

В этом проекте ты будешь передавать токен через переменную окружения `BOT_TOKEN`:

```bash
BOT_TOKEN="1234567890:ABCdefGhIJKlmNoPqRsTuVwXyZ" go run .
```

## Ядро 4: Telegram Bot API в общих чертах

Все методы Telegram Bot API — это HTTP-запросы по адресу:

```
https://api.telegram.org/bot<TOKEN>/<METHOD>
```

Например:

```
https://api.telegram.org/bot1234567890:ABC.../getMe
https://api.telegram.org/bot1234567890:ABC.../sendMessage
https://api.telegram.org/bot1234567890:ABC.../getUpdates
```

Запросы могут быть `GET` или `POST` (Telegram принимает оба). Параметры можно передавать:

- В query string (`?chat_id=12345&text=hello`)
- В теле запроса как `application/x-www-form-urlencoded`
- В теле как `application/json`

Ответ всегда в JSON и имеет одинаковую структуру:

**Успех:**
```json
{
  "ok": true,
  "result": { ... }
}
```

**Ошибка:**
```json
{
  "ok": false,
  "error_code": 401,
  "description": "Unauthorized"
}
```

Поле `ok` — главный индикатор. Сначала проверяешь его, потом разбираешь `result` или ошибку.

## Ядро 5: метод getMe

Это самый простой метод API: «расскажи о себе, кто ты вообще такой бот». Не требует параметров.

```
GET https://api.telegram.org/bot<TOKEN>/getMe
```

Ответ:

```json
{
  "ok": true,
  "result": {
    "id": 1234567890,
    "is_bot": true,
    "first_name": "My Task Bot",
    "username": "lev_todo_bot",
    "can_join_groups": true,
    "can_read_all_group_messages": false,
    "supports_inline_queries": false
  }
}
```

Это идеальный «hello world»: проверяет, что:

1. Сетевое подключение к Telegram работает
2. Токен валидный
3. Ты правильно понял формат HTTP-запросов и JSON-ответов

В этом модуле ты вызовешь именно `getMe`. Никаких сообщений, никаких пользователей — только проверка.

## Ядро 6: HTTP-клиент в Go

В Проекте 4 ты использовал `net/http` для написания **сервера**. Тот же пакет умеет делать **клиентские** запросы:

```go
import "net/http"

resp, err := http.Get("https://api.telegram.org/bot.../getMe")
if err != nil {
    return err
}
defer resp.Body.Close()

// Прочитать тело ответа
data, err := io.ReadAll(resp.Body)
if err != nil {
    return err
}

// Распарсить JSON
var result struct {
    Ok     bool            `json:"ok"`
    Result json.RawMessage `json:"result"`
}
if err := json.Unmarshal(data, &result); err != nil {
    return err
}
```

Несколько важных моментов:

1. **`http.Get(url)`** — самый простой способ. Возвращает `*http.Response`. Внутри использует `http.DefaultClient`.

2. **`defer resp.Body.Close()`** — **обязательно**, как `defer rows.Close()` в БД. `Body` — это поток, который держит соединение. Без `Close` соединение не вернётся в пул.

3. **`io.ReadAll(resp.Body)`** — читает всё тело сразу в `[]byte`. Подходит для маленьких ответов (а ответы Telegram — маленькие). Для больших ответов — стриминг.

4. **`json.RawMessage`** — это `[]byte`, который можно распарсить **позже**. Удобно, когда ты сначала проверяешь `ok`, и только потом разбираешься с конкретным форматом результата.

## Ядро 7: лучше — http.Client с таймаутом

`http.Get` и `http.DefaultClient` — это **глобальный клиент без таймаута**. Если Telegram тормозит — твой запрос будет висеть **бесконечно**. Это плохо для долгоживущего бота.

Правильный подход — создать **свой клиент** с настройками:

```go
client := &http.Client{
    Timeout: 30 * time.Second,
}

resp, err := client.Get("https://api.telegram.org/bot.../getMe")
```

Или ещё лучше — не использовать `http.Get`, а собирать запрос вручную через `http.NewRequest`:

```go
req, err := http.NewRequest("GET", url, nil)
if err != nil {
    return err
}

resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close()
```

Это даёт контроль над:

- HTTP-методом (GET/POST/PUT)
- Заголовками (User-Agent, Content-Type)
- Телом запроса
- **Контекстом** (через `req = req.WithContext(ctx)`)

В модуле 1 мы делаем простой `client.Get`. В модулях 2-3 перейдём на `Request`.

## Ядро 8: структура нашего «клиента» Telegram

В этом проекте мы пишем **тонкую обёртку** над Telegram Bot API. Без библиотек, только `net/http` + `encoding/json`.

Структура:

```go
type TelegramClient struct {
    token  string
    client *http.Client
    apiURL string
}

func NewTelegramClient(token string) *TelegramClient {
    return &TelegramClient{
        token:  token,
        client: &http.Client{Timeout: 30 * time.Second},
        apiURL: "https://api.telegram.org/bot" + token,
    }
}
```

И методы будут добавляться по мере роста проекта:

- Модуль 1: `GetMe()`
- Модуль 2: `GetUpdates(offset)`, `SendMessage(chatID, text)`
- ... и так далее

Этот тип — **наш собственный** «mini telegram-bot-api». В конце проекта ты увидишь, что весь Telegram Bot API — это десяток таких методов плюс типы данных. Никакой магии, всё прозрачно.

## Полный пример модуля 1

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
    "os"
    "time"
)

type TelegramClient struct {
    token  string
    client *http.Client
    apiURL string
}

func NewTelegramClient(token string) *TelegramClient {
    return &TelegramClient{
        token:  token,
        client: &http.Client{Timeout: 30 * time.Second},
        apiURL: "https://api.telegram.org/bot" + token,
    }
}

type User struct {
    ID        int64  `json:"id"`
    IsBot     bool   `json:"is_bot"`
    FirstName string `json:"first_name"`
    Username  string `json:"username"`
}

type apiResponse struct {
    Ok          bool            `json:"ok"`
    Result      json.RawMessage `json:"result"`
    ErrorCode   int             `json:"error_code"`
    Description string          `json:"description"`
}

func (c *TelegramClient) GetMe() (*User, error) {
    resp, err := c.client.Get(c.apiURL + "/getMe")
    if err != nil {
        return nil, fmt.Errorf("http get: %w", err)
    }
    defer resp.Body.Close()

    data, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("read body: %w", err)
    }

    var apiResp apiResponse
    if err := json.Unmarshal(data, &apiResp); err != nil {
        return nil, fmt.Errorf("unmarshal: %w", err)
    }

    if !apiResp.Ok {
        return nil, fmt.Errorf("telegram error %d: %s", apiResp.ErrorCode, apiResp.Description)
    }

    var user User
    if err := json.Unmarshal(apiResp.Result, &user); err != nil {
        return nil, fmt.Errorf("unmarshal result: %w", err)
    }

    return &user, nil
}

func main() {
    token := os.Getenv("BOT_TOKEN")
    if token == "" {
        log.Fatal("BOT_TOKEN не задан")
    }

    client := NewTelegramClient(token)

    me, err := client.GetMe()
    if err != nil {
        log.Fatalf("getMe: %v", err)
    }

    log.Printf("Бот авторизован: id=%d username=@%s name=%q",
        me.ID, me.Username, me.FirstName)
}
```

Запуск:

```bash
BOT_TOKEN="1234567890:ABC..." go run .
```

Вывод:

```
2026/04/13 12:00:00 Бот авторизован: id=1234567890 username=@lev_todo_bot name="My Task Bot"
```

Если токен правильный и бот существует — увидишь именно это. Если нет — понятную ошибку.

## Вопросы на понимание

1. Что такое Telegram-бот с точки зрения Telegram?
2. Кто такой @BotFather и зачем он нужен?
3. Что такое **токен** и почему его нельзя коммитить?
4. Какой URL у Telegram Bot API?
5. Какая структура у ответа Telegram (поля верхнего уровня)?
6. Что делает метод `getMe` и зачем он полезен в начале работы?
7. Почему **обязательно** делать `defer resp.Body.Close()`?
8. Почему нельзя использовать `http.DefaultClient` для long-running ботов?
9. Что такое `json.RawMessage` и в чём её польза?
