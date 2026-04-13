# Модуль 2. Long polling и echo-бот — подсказки

> ⚠️ Открывай только после 30 минут самостоятельной борьбы.

## Уровень 1: наводящие вопросы

1. Какой метод API возвращает новые сообщения?
2. Какой параметр контролирует, какие обновления возвращать?
3. Как обновлять `offset` после обработки?
4. Какой метод отправляет сообщение?
5. Откуда брать `chat_id` для ответа?

## Уровень 2: про типы Update / Message / Chat

Минимальные определения:

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

- **`*Message`** — указатель, потому что `message` опционально (Update может содержать другие типы данных)
- **`User`** — тот же, что в модуле 1 (можешь переиспользовать)
- **`Chat.ID`** — это и есть то, что передаётся в `sendMessage` как `chat_id`

## Уровень 3: про GetUpdates

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
        return nil, fmt.Errorf("unmarshal: %w", err)
    }

    return updates, nil
}
```

Заметь: `Result` — это **JSON-массив**, поэтому `Unmarshal` в `[]Update`, не в `Update`.

## Уровень 4: про SendMessage

```go
type sendMessageRequest struct {
    ChatID int64  `json:"chat_id"`
    Text   string `json:"text"`
}

func (c *TelegramClient) SendMessage(chatID int64, text string) error {
    body := sendMessageRequest{ChatID: chatID, Text: text}
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

Несколько моментов:

- **`http.Post`** принимает три аргумента: URL, content-type, тело
- **`bytes.NewReader(data)`** превращает `[]byte` в `io.Reader`
- **`Content-Type: application/json`** обязателен для JSON-тела
- Не возвращаем результат отправленного сообщения (можешь добавить, если хочешь)

## Уровень 5: про правильный таймаут http.Client

**Главная ловушка модуля.** Если `http.Client.Timeout = 30 секунд` и ты вызываешь `getUpdates(timeout=30)` — клиент порвёт соединение **раньше**, чем Telegram успеет ответить.

Решение — клиент должен иметь таймаут **больше**:

```go
client: &http.Client{Timeout: 60 * time.Second}
```

Или вообще без таймаута:

```go
client: &http.Client{}  // Timeout: 0 = бесконечно
```

Для production правильнее использовать **разные клиенты** для разных типов запросов:
- Один с таймаутом 60s — для long polling
- Один с таймаутом 10s — для всех остальных методов

Для нашего проекта — один клиент с 60s достаточно.

## Уровень 6: про правильное обновление offset

```go
var offset int64 = 0

for {
    updates, err := bot.GetUpdates(offset)
    if err != nil {
        log.Printf("getUpdates error: %v", err)
        time.Sleep(5 * time.Second)
        continue
    }

    for _, upd := range updates {
        // обработать сообщение
        if upd.Message != nil && upd.Message.Text != "" {
            handleMessage(bot, upd.Message)
        }

        // обновить offset
        if upd.UpdateID >= offset {
            offset = upd.UpdateID + 1
        }
    }
}
```

Несколько важных вещей:

1. **`offset` объявлен ДО цикла** — иначе он сбрасывается на каждой итерации
2. **`offset = upd.UpdateID + 1`** — следующий getUpdates вернёт обновления **после** этого id
3. **Проверка `upd.UpdateID >= offset`** — на случай, если обновления пришли «не по порядку» (у Telegram такого почти не бывает, но защита не помешает)

## Уровень 7: про обработку nil-полей

```go
for _, upd := range updates {
    // Update может не иметь Message (например, callback_query)
    if upd.Message == nil {
        continue
    }
    // Message может не иметь Text (например, фото)
    if upd.Message.Text == "" {
        continue
    }
    // Только теперь — обработка
    handleMessage(bot, upd.Message)
}
```

В Telegram API много типов сообщений, и не все они имеют текст. Если ты обращаешься к `upd.Message.Text` без проверок — упадёшь с nil pointer dereference на первом же необычном сообщении (стикер, голосовое, фото).

## Уровень 8: про восстановление после ошибок

```go
for {
    updates, err := bot.GetUpdates(offset)
    if err != nil {
        log.Printf("getUpdates error: %v", err)
        time.Sleep(5 * time.Second)
        continue  // ← важно: continue, не break и не return
    }
    // ...
}
```

`continue` — следующая итерация. Бот **не падает** при ошибке, не выходит из цикла. Просто логирует, ждёт и пробует снова.

Без `time.Sleep` ты будешь спамить Telegram запросами в бесконечном цикле, если они все падают (например, токен временно невалиден). Это плохо: тебя могут забанить.

## Уровень 9: про bytes.NewReader

`http.Post` принимает `io.Reader` для тела запроса. У тебя `[]byte` (от `json.Marshal`). Конвертация:

```go
import "bytes"

bytes.NewReader(data)  // []byte → io.Reader
```

Альтернативы:
- `strings.NewReader(string(data))` — то же самое через строку
- `bytes.NewBuffer(data)` — тоже работает, но `Reader` чуть эффективнее

Для нашего модуля — `bytes.NewReader` стандартный выбор.

## Если совсем тупик

Скажи Claude конкретно: «модуль 2 проекта 6, [конкретная проблема]». Самые частые ситуации:

- **«Бот отвечает много раз на одно сообщение»** → не обновляешь offset
- **«Бот не реагирует»** → проверь логи: получает ли он `Updates`? Может быть, забыл вызвать `SendMessage`
- **«context deadline exceeded»** → http.Client с слишком маленьким таймаутом
- **«panic: runtime error: invalid memory address»** → не проверил `upd.Message != nil`
- **«Telegram error 400: chat not found»** → используешь неправильный `chat_id`, проверь, что берёшь из `msg.Chat.ID`
- **«i/o timeout»** → проблема с интернетом или Telegram заблокирован — нужен VPN
