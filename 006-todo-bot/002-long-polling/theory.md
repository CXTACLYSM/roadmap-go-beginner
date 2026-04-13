# Модуль 2. Long polling и echo-бот — теория

## Контекст

В модуле 1 ты вызвал `getMe` и убедился, что инфраструктура работает. Теперь начинается **настоящий бот**: он будет получать сообщения от пользователей и отвечать на них.

В этом модуле:

- Поймёшь, как Telegram доставляет тебе входящие сообщения
- Реализуешь два главных метода: `getUpdates` и `sendMessage`
- Напишешь первый **long polling loop**
- Сделаешь **echo-бота** — на любое входящее сообщение бот ответит тем же текстом

После этого модуля ты сможешь сказать: «у меня есть рабочий бот».

## Ядро 1: как боты получают сообщения

Telegram даёт **два способа** доставки входящих сообщений боту:

### Способ 1: Webhooks (push)

Ты регистрируешь URL твоего сервера. Когда приходит новое сообщение — Telegram сам **присылает** HTTP POST на этот URL.

**Плюсы:** мгновенно, нет лишних запросов.
**Минусы:** требует **публичный HTTPS**-сервер с валидным сертификатом. Невозможно использовать локально без туннеля (ngrok, cloudflared).

### Способ 2: Long polling (pull)

Твоя программа **сама** периодически спрашивает Telegram «есть для меня сообщения?». Если есть — получает их сразу. Если нет — Telegram **держит соединение открытым** до 30-60 секунд, ждёт новых сообщений, и только тогда отвечает (либо пустым массивом, либо реальными данными).

**Плюсы:** работает с любым устройством, не требует публичного URL.
**Минусы:** чуть менее эффективно (хотя long polling практически так же быстр, как webhooks).

**Для нашего проекта используем long polling.** Это:

- Можно тестировать локально (на твоём ноутбуке)
- Не требует деплоя
- Позволяет понять «что под капотом»
- В реальных продакшен-ботах тоже широко используется (особенно для маленьких сервисов)

## Ядро 2: метод getUpdates

Запрос:

```
GET https://api.telegram.org/bot<TOKEN>/getUpdates?offset=0&timeout=30
```

Параметры:

- **`offset`** — id последнего обновления, которое ты **уже обработал**. Telegram вернёт обновления **после** этого id. По умолчанию 0 — вернёт всё с начала.
- **`timeout`** — long polling таймаут в секундах. Если 0 — вернётся сразу (классический polling). Если 30 — Telegram **подождёт** до 30 секунд и вернёт либо сообщения, либо пустой массив.
- **`limit`** — максимум обновлений за один ответ (1-100, по умолчанию 100).

Ответ:

```json
{
  "ok": true,
  "result": [
    {
      "update_id": 12345,
      "message": {
        "message_id": 67,
        "from": { "id": 999, "first_name": "Lev", "username": "..." },
        "chat": { "id": 999, "type": "private" },
        "date": 1712880000,
        "text": "привет"
      }
    },
    {
      "update_id": 12346,
      "message": { ... }
    }
  ]
}
```

`result` — это **массив** объектов `Update`. Каждый имеет `update_id` и одно из полей с данными (чаще всего `message`).

## Ядро 3: что такое offset и зачем он нужен

**Это критическая концепция, которую путают все новички.**

`getUpdates` возвращает обновления, которые **ты ещё не обработал**. Но Telegram сам не знает, что ты обработал, а что нет. **Ты должен сам сказать ему**, передав параметр `offset`.

Правило: **`offset` = `last_update_id + 1`**.

То есть: «дай мне обновления, у которых `update_id` строго больше, чем `last_update_id`».

Сценарий:

1. Ты вызвал `getUpdates(offset=0)` — получил обновления с `update_id` 100, 101, 102.
2. Обработал их.
3. **Следующий вызов**: `getUpdates(offset=103)`.
4. Telegram вернёт только обновления с `update_id >= 103`.
5. Получил `103, 104` → следующий вызов `offset=105`.

**Что произойдёт, если забыть менять offset:**

```go
// ЛОВУШКА
for {
    updates := bot.GetUpdates(0)  // offset всегда 0
    for _, upd := range updates {
        handle(upd)
    }
}
```

Каждый вызов вернёт **те же самые** обновления. Бот будет обрабатывать одно и то же сообщение бесконечно. Это типичная ошибка модуля 2.

**Правильно:**

```go
offset := 0
for {
    updates, _ := bot.GetUpdates(offset)
    for _, upd := range updates {
        handle(upd)
        if upd.UpdateID >= offset {
            offset = upd.UpdateID + 1
        }
    }
}
```

После обработки последнего обновления `offset` сдвигается на следующее, и Telegram больше не отдаёт уже обработанные.

## Ядро 4: что такое long polling

Когда ты передаёшь `timeout=30`, происходит следующее:

1. Твоя программа отправляет HTTP-запрос
2. Telegram получает запрос, проверяет «есть ли новые сообщения для этого бота?»
3. **Если есть** — сразу возвращает их
4. **Если нет** — **держит соединение открытым** и ждёт до 30 секунд
5. Если за 30 секунд приходит сообщение — возвращает его. Если нет — возвращает пустой массив.

Для **твоего** Go-кода это выглядит как «обычный HTTP-запрос, который иногда долго отвечает». Но за этим стоит сильная оптимизация: вместо того, чтобы спамить запросами каждую секунду, ты отправляешь один запрос на 30 секунд и получаешь сообщения **почти мгновенно**, как только они появляются.

**Это критично для http.Client:** твой клиент должен иметь таймаут **больше**, чем `timeout` параметра. Если ты ставишь `Client.Timeout = 30s` и `getUpdates(timeout=30)` — иногда будет срабатывать таймаут клиента **до** того, как Telegram отдаст ответ. Решение: либо `Client.Timeout = 0` (без таймаута), либо `Client.Timeout = 60s` для запаса.

## Ядро 5: метод sendMessage

Самый важный метод для отправки ответа:

```
POST https://api.telegram.org/bot<TOKEN>/sendMessage
Content-Type: application/json

{
  "chat_id": 999,
  "text": "Привет!"
}
```

Параметры:

- **`chat_id`** — id чата, куда отправлять. Берётся из `update.message.chat.id`.
- **`text`** — текст сообщения. Максимум 4096 символов.
- **`parse_mode`** — `Markdown`, `HTML` или ничего (plain text). Влияет на форматирование.
- **`reply_to_message_id`** — id сообщения, на которое отвечаешь (создаст «ответ»-цитату).
- ... и десятки других параметров для клавиатур, медиа, и т.д.

Для модуля 2 нужны только `chat_id` и `text`.

## Ядро 6: реализация sendMessage в нашем клиенте

```go
type SendMessageRequest struct {
    ChatID int64  `json:"chat_id"`
    Text   string `json:"text"`
}

func (c *TelegramClient) SendMessage(chatID int64, text string) error {
    body := SendMessageRequest{ChatID: chatID, Text: text}
    data, err := json.Marshal(body)
    if err != nil {
        return fmt.Errorf("marshal: %w", err)
    }

    resp, err := c.client.Post(
        c.apiURL+"/sendMessage",
        "application/json",
        bytes.NewReader(data),
    )
    if err != nil {
        return fmt.Errorf("post: %w", err)
    }
    defer resp.Body.Close()

    var apiResp apiResponse
    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return fmt.Errorf("decode: %w", err)
    }

    if !apiResp.Ok {
        return fmt.Errorf("telegram error %d: %s", apiResp.ErrorCode, apiResp.Description)
    }

    return nil
}
```

Несколько новых вещей:

1. **`http.Post`** вместо `http.Get`. Принимает URL, content-type и тело.
2. **`bytes.NewReader(data)`** — превращает `[]byte` в `io.Reader` (как требует `Post`).
3. **`json.Marshal` для тела запроса** — собираем JSON из структуры.
4. **Не возвращаем результат** — `sendMessage` возвращает отправленное сообщение, но обычно нам это не нужно.

## Ядро 7: типы для Update и Message

Для парсинга `getUpdates` нам нужны Go-структуры, соответствующие JSON Telegram:

```go
type Update struct {
    UpdateID int64    `json:"update_id"`
    Message  *Message `json:"message,omitempty"`
}

type Message struct {
    MessageID int64  `json:"message_id"`
    From      *User  `json:"from,omitempty"`
    Chat      Chat   `json:"chat"`
    Date      int64  `json:"date"`
    Text      string `json:"text,omitempty"`
}

type Chat struct {
    ID   int64  `json:"id"`
    Type string `json:"type"`
}
```

Заметь:

- **`*Message`** (указатель) — потому что `message` опционально в `Update`. Update может нести не сообщение, а, например, callback query (это для другой задачи).
- **`Chat.ID`** — это и есть `chat_id`, который мы передаём в `sendMessage`.
- **`omitempty`** — поле не сериализуется, если пустое. Полезно при создании запросов.

## Ядро 8: реализация GetUpdates

```go
func (c *TelegramClient) GetUpdates(offset int64) ([]Update, error) {
    url := fmt.Sprintf("%s/getUpdates?offset=%d&timeout=30", c.apiURL, offset)

    resp, err := c.client.Get(url)
    if err != nil {
        return nil, fmt.Errorf("get: %w", err)
    }
    defer resp.Body.Close()

    var apiResp apiResponse
    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return nil, fmt.Errorf("decode: %w", err)
    }

    if !apiResp.Ok {
        return nil, fmt.Errorf("telegram error %d: %s", apiResp.ErrorCode, apiResp.Description)
    }

    var updates []Update
    if err := json.Unmarshal(apiResp.Result, &updates); err != nil {
        return nil, fmt.Errorf("unmarshal updates: %w", err)
    }

    return updates, nil
}
```

Заметь: те же шаги, что в `GetMe` — Get → Close → Decode → проверка `Ok` → парсинг `Result`.

⚠️ **Тонкость:** клиент должен иметь **таймаут больше** 30 секунд (long polling timeout). Иначе клиент порвёт соединение раньше, чем Telegram ответит. Установи `Client.Timeout = 60 * time.Second` (или `0` для бесконечности — тоже валидно для long polling).

## Ядро 9: главный цикл бота

```go
func main() {
    token := os.Getenv("BOT_TOKEN")
    if token == "" {
        log.Fatal("BOT_TOKEN не задан")
    }

    bot := NewTelegramClient(token)

    me, err := bot.GetMe()
    if err != nil {
        log.Fatalf("getMe: %v", err)
    }
    log.Printf("Бот запущен: @%s", me.Username)

    var offset int64 = 0
    for {
        updates, err := bot.GetUpdates(offset)
        if err != nil {
            log.Printf("getUpdates error: %v", err)
            time.Sleep(5 * time.Second)
            continue
        }

        for _, upd := range updates {
            if upd.Message != nil && upd.Message.Text != "" {
                handleMessage(bot, upd.Message)
            }

            if upd.UpdateID >= offset {
                offset = upd.UpdateID + 1
            }
        }
    }
}

func handleMessage(bot *TelegramClient, msg *Message) {
    log.Printf("Сообщение от @%s: %s", msg.From.Username, msg.Text)

    reply := "Ты сказал: " + msg.Text
    if err := bot.SendMessage(msg.Chat.ID, reply); err != nil {
        log.Printf("sendMessage error: %v", err)
    }
}
```

Структура:

1. Бесконечный цикл `for { ... }`
2. Внутри — `getUpdates(offset)`
3. Если ошибка — лог, пауза, продолжить (не падать)
4. Для каждого обновления — обработать сообщение и обновить offset
5. **Между итерациями нет `Sleep`** — long polling сам делает паузу

Это **минимальный жизнеспособный бот**. Запусти, напиши ему — он эхом ответит. Это твой first working bot.

## Ядро 10: восстановление после ошибок

В долгоживущих программах **сетевые ошибки неизбежны**:

- Интернет на ноутбуке моргнул
- Telegram временно недоступен
- Прокси проглотил пакет

Программа **не должна падать** от этих ошибок. Правильное поведение:

```go
updates, err := bot.GetUpdates(offset)
if err != nil {
    log.Printf("getUpdates error: %v", err)
    time.Sleep(5 * time.Second)
    continue
}
```

- Залогировали ошибку
- **Подождали** немного (чтобы не спамить Telegram сразу же)
- Продолжили цикл

5 секунд — разумная задержка для retry. В более продвинутых ботах используют **exponential backoff** (1с, 2с, 4с, 8с, ... до некоторого максимума), но для нашего проекта 5 секунд достаточно.

## Вопросы на понимание

1. Какие два способа Telegram доставляет сообщения боту? Какой выбрали и почему?
2. Что такое **`offset`** в `getUpdates` и зачем он нужен?
3. Что произойдёт, если **забыть менять offset**?
4. Что такое **long polling**? Чем оно отличается от обычного polling?
5. Какой `Timeout` нужен для `http.Client`, если `getUpdates(timeout=30)`?
6. Какие два главных метода API мы реализуем в этом модуле?
7. Из какого поля брать `chat_id` для ответа на сообщение?
8. Что должен делать бот, если `getUpdates` вернул ошибку — упасть или продолжить?
9. Зачем после ошибки **`time.Sleep`**, а не сразу retry?
