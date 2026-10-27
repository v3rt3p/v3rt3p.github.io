# API function/state-сервера

Processor использует HTTP-сервисы для состояния и функций. Их базовые URL задаются через `PROCESSOR_STATE_SERVER_URLS` и `PROCESSOR_FUNCTION_SERVER_URLS`.

Общие схемы запросов и ответов находятся в [`v3rt3p/types/src/function-state-server.ts`](https://github.com/v3rt3p/types/blob/master/src/function-state-server.ts).

## State endpoints

### `GET /state`

Возвращает состояние, не зависящее от текущей сессии или устройства.

```json
{
  "temperature": {
    "description": "Температура снаружи",
    "value": "12 °C"
  }
}
```

Ответ — объект с ключами состояния. Каждый элемент содержит понятные человеку `description` и `value`.

### `PUT /state`

Возвращает состояние, зависящее от сессии или устройства.

Запрос:

```json
{
  "sessionId": "processor-session-id",
  "metadata": {
    "deviceId": "station-123"
  }
}
```

Ответ:

```json
{
  "room": {
    "description": "Текущая комната",
    "value": "кухня"
  }
}
```

`metadata` содержит данные интеграции. `sessionId` идентифицирует WebSocket-сессию processor-а; относитесь к нему как к непрозрачному идентификатору.

## Function endpoints

### `GET /functions`

Возвращает функции, не зависящие от текущей сессии или устройства.

```json
{
  "smart_speaker_tts": {
    "arguments": {
      "type": "object",
      "properties": {
        "text": { "type": "string" }
      },
      "required": ["text"]
    },
    "description": "Произнести текст на станции",
    "hasResponse": false
  }
}
```

`arguments` — JSON Schema аргументов функции. `hasResponse` показывает, должен ли processor ждать результат.

### `PUT /functions`

Возвращает функции, зависящие от текущей сессии или устройства. Запрос имеет тот же формат, что и `PUT /state`, с `sessionId` и `metadata`.

Ответ имеет тот же формат, что и `GET /functions`.

### `POST /functions`

Выполняет вызов функции:

```json
{
  "sessionId": "processor-session-id",
  "metadata": {
    "deviceId": "station-123"
  },
  "name": "smart_speaker_tts",
  "arguments": {
    "text": "Включаю свет"
  }
}
```

Ответ, если функция возвращает данные:

```json
{
  "response": {
    "ok": true
  }
}
```

Поле `response` необязательно. Для функций с `hasResponse: false` возвращайте, например, `{}`.

## Заметки по реализации

Processor сначала получает независимые state/functions через `GET`, затем запрашивает зависимые значения через `PUT` и вызывает функции через `POST`. Function-серверы должны принимать JSON и возвращать JSON. Сервис должен быть доступен из контейнера processor, а не только через loopback хоста.

Quasar предоставляет совместимый function/state-сервер на внутреннем function-state-порту, когда эта функция включена. См. [HTTP API Quasar](../quasar-server/http-api.md).
