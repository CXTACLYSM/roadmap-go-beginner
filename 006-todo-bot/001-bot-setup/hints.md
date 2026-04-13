# Модуль 1. Регистрация и токен — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Кто такой @BotFather и какая команда создаёт нового бота?
2. Какой URL у Telegram Bot API?
3. Какой пакет Go умеет делать HTTP-запросы?
4. Какая структура у ответа Telegram (поля верхнего уровня)?
5. Зачем нужен `defer resp.Body.Close()`?

## Уровень 2: про BotFather

`@BotFather` — это бот в Telegram. Найди его поиском (с символом `@`). Должна быть синяя галочка верификации.

Команды:
- `/newbot` — создать нового
- `/mybots` — список твоих
- `/help` — все команды

При создании:

1. Имя — любое, например, «Lev Todo Bot». Это отображается в профиле.
2. Username — обязательно оканчивается на `bot` или `Bot`. Например, `lev_todo_bot`. Username должен быть **уникальным** во всём Telegram, поэтому простые имена обычно заняты.
3. После успеха @BotFather пришлёт сообщение с токеном вида `1234567890:ABCdefGhIJKlmNoPqRsTuVwXyZ` и предупреждение «Keep your token secure».

Если username занят — пробуй варианты с цифрами или подчёркиваниями.

## Уровень 3: про сохранение токена

**Никогда** не пиши токен прямо в коде:

```go
// КАТЕГОРИЧЕСКИ НЕЛЬЗЯ
token := "1234567890:ABCdefGhIJKlmNoPqRsTuVwXyZ"
```

Варианты для разработки:

**A) Через переменную окружения при запуске**
```bash
BOT_TOKEN="..." go run .
```

**B) Через файл, который в .gitignore**
```bash
echo "BOT_TOKEN=..." > .env
echo ".env" >> .gitignore
```
И загружать через `godotenv` или вручную.

**C) Через локальный файл вне репозитория**
```bash
echo "1234567890:ABC..." > ~/.todo_bot_token
chmod 600 ~/.todo_bot_token
```
И:
```bash
BOT_TOKEN="$(cat ~/.todo_bot_token)" go run .
```

Для модуля 1 — вариант **A** или **C**. Они простейшие и безопасные.

## Уровень 4: про http.Client с таймаутом

Не используй `http.Get(...)` напрямую — он использует `DefaultClient` без таймаута:

```go
// ПЛОХО
resp, err := http.Get(url)
```

Лучше — свой клиент с таймаутом:

```go
client := &http.Client{
    Timeout: 30 * time.Second,
}
resp, err := client.Get(url)
```

Без таймаута, если сервер Telegram зависнет, твой запрос будет висеть **бесконечно**. Для долгоживущего бота это смерть.

30 секунд — разумное значение для обычных запросов. Для long polling (модуль 2) понадобится больше.

## Уровень 5: про парсинг JSON-ответа

Telegram возвращает JSON вида:

```json
{
  "ok": true,
  "result": { "id": 123, "username": "..." }
}
```

или

```json
{
  "ok": false,
  "error_code": 401,
  "description": "Unauthorized"
}
```

Парсить можно одной структурой через `json.RawMessage`:

```go
type apiResponse struct {
    Ok          bool            `json:"ok"`
    Result      json.RawMessage `json:"result"`
    ErrorCode   int             `json:"error_code"`
    Description string          `json:"description"`
}
```

`json.RawMessage` — это `[]byte`, который **не парсится** на этом этапе. Ты сначала проверяешь `Ok`, и только если успех — парсишь `Result` в конкретный тип.

```go
var apiResp apiResponse
if err := json.Unmarshal(data, &apiResp); err != nil {
    return nil, err
}

if !apiResp.Ok {
    return nil, fmt.Errorf("telegram error %d: %s", apiResp.ErrorCode, apiResp.Description)
}

var user User
if err := json.Unmarshal(apiResp.Result, &user); err != nil {
    return nil, err
}
```

Это удобно, потому что разные методы возвращают разные типы в `Result`, и одну общую структуру `apiResponse` можно переиспользовать.

## Уровень 6: про io.ReadAll vs json.NewDecoder

Два способа прочитать тело ответа:

**Способ A: `io.ReadAll` + `json.Unmarshal`**
```go
data, err := io.ReadAll(resp.Body)
if err != nil { ... }
json.Unmarshal(data, &result)
```

**Способ B: `json.NewDecoder + Decode`**
```go
err := json.NewDecoder(resp.Body).Decode(&result)
```

Оба работают. Для **маленьких** ответов разницы нет. Способ A позволяет в случае ошибки вывести **сырое тело** в лог:

```go
data, _ := io.ReadAll(resp.Body)
log.Printf("ответ Telegram: %s", string(data))
```

— это полезно для отладки. В нашем модуле — выбирай любой.

## Уровень 7: про сборку URL

```go
url := c.apiURL + "/getMe"
```

где `c.apiURL = "https://api.telegram.org/bot" + token`. Простая конкатенация работает, но **только** для методов без параметров.

Когда параметры будут (`sendMessage`, `getUpdates`), уже нужен `net/url` или формирование query-строки. Это в модуле 2.

## Уровень 8: про сообщения об ошибках Telegram

Самые частые:

- **`401 Unauthorized`** — токен неправильный или отозван. Перевыпусти через `/token` в @BotFather.
- **`404 Not Found`** — неправильный URL (опечатка в названии метода).
- **`400 Bad Request: chat not found`** — `chat_id` неправильный (для `sendMessage`). В модуле 1 не встретишь.

Все ошибки приходят в одном формате — `{"ok": false, "error_code": ..., "description": ...}`. Главное — обработать их одинаково.

## Уровень 9: про defer resp.Body.Close()

Без этого:

- HTTP-соединение остаётся открытым в пуле
- Через много таких запросов файловые дескрипторы кончаются
- Программа начинает падать «странными» ошибками

`defer resp.Body.Close()` — обязательная привычка **сразу** после успешного HTTP-запроса. Как `defer rows.Close()` в БД.

⚠️ **Тонкость:** делай `defer Close()` **только** если `err == nil`. Если HTTP-запрос провалился, `resp` может быть `nil`, и `Close()` упадёт. Правильно:

```go
resp, err := client.Get(url)
if err != nil {
    return nil, err
}
defer resp.Body.Close()
```

`defer` идёт **после** проверки ошибки, не до.

## Если совсем тупик

Скажи Claude конкретно: «модуль 1 проекта 6, [конкретная проблема]». Самые частые ситуации:

- **«Telegram error 401»** → токен неправильный или с лишними символами в `BOT_TOKEN`
- **«no such host»** → проблема с DNS или интернетом, или Telegram заблокирован у тебя — нужен VPN
- **«i/o timeout»** → тоже проблемы с сетью или Telegram недоступен
- **«unmarshal: invalid character»** → ответ Telegram в другом формате (HTML вместо JSON), скорее всего ты получаешь страницу ошибки от прокси/Cloudflare
