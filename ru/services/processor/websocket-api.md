# WebSocket API processor-а

Processor принимает одно WebSocket-подключение на диалог:

```text
ws://processor:8080/process
```

Путь подключения — `/process`. Сообщения передаются как JSON text frames. Общие схемы находятся в [`v3rt3p/types/src/processor.ts`](https://github.com/v3rt3p/types/blob/master/src/processor.ts).

## Порядок сообщений

1. Откройте `/process`.
2. Отправьте одно сообщение `prepare` с functions, доступными текущему устройству.
3. Отправьте одно или несколько сообщений `process` для пользовательского ввода.
4. Читайте сообщения `partialResponse`, пока `finished` не станет `true`.
5. Если `shouldListen` равен `true`, оставьте диалог открытым и отправьте следующее `process`. Иначе сервер закроет WebSocket после ответа.

## Сообщения клиента

### `prepare`

Загружает базовый prompt, независимое состояние и функции сессии.

```json
{
  "type": "prepare",
  "data": {
    "directiveFunctions": {
      "smart_speaker_tts": {
        "arguments": {
          "type": "object",
          "properties": {
            "text": { "type": "string" }
          }
        },
        "description": "Произнести текст на текущей станции"
      }
    }
  }
}
```

`directiveFunctions` — map имени функции на её JSON Schema и описание. Processor добавляет эти функции в prompt; сам он не выполняет их, если они не подключены через настроенный function- или MCP-сервис.

### `process`

Обрабатывает одну распознанную фразу:

```json
{
  "type": "process",
  "data": {
    "text": "Какая сегодня погода?",
    "metadata": {
      "internal": {
        "deviceId": "station-123"
      },
      "readyToUse": {
        "room": {
          "description": "Текущая комната",
          "value": "кухня"
        }
      }
    }
  }
}
```

- `text` — распознанный пользовательский текст;
- `metadata.internal` передаётся function/state-интеграциям и не должен показываться как пользовательское состояние;
- `metadata.readyToUse` содержит состояние, уже подготовленное для LLM.

## Сообщения сервера

### `partialResponse`

Processor отправляет ответ после каждого шага LLM или tool:

```json
{
  "type": "partialResponse",
  "data": {
    "directives": [],
    "text": "Сегодня солнечно.",
    "finished": true,
    "shouldListen": false
  }
}
```

| Поле | Значение |
| --- | --- |
| `text` | Текст для произнесения или отображения. |
| `directives` | Directives устройства, возвращённые моделью; каждый элемент содержит `name` и `arguments`. |
| `finished` | `false`, пока продолжается обработка или вызов tools; `true`, когда запрос завершён. |
| `shouldListen` | Присутствует при `finished: true`; указывает, нужно ли ждать следующую фразу. |

Если модель вызывает функцию, которая должна вернуть результат, processor сначала отправляет `finished: false`, выполняет функцию, а затем отправляет новый `partialResponse`. Функция без ответа может присутствовать в `directives`, но не удерживает запрос открытым.

## Ошибки и разрыв соединения

Неверный JSON или сообщение, не соответствующее схеме, отклоняется обработчиком сообщений processor-а. Сервис пишет ошибку WebSocket в лог и закрывает сессию. Разрыв до `finished: true` считайте ошибкой запроса; повторяйте его только если станция безопасно повторит фразу.

Протокол реализован в `src/main.ts` и `src/processor.ts` репозитория processor. При изменении полей сверяйте страницу с зафиксированной схемой `v3rt3p/types`.
