# HTTP API Quasar

Quasar предоставляет API администрирования на `API_PORT` — `31116` в legacy Compose:

```text
http://quasar-server:31116
```

Интерактивная OpenAPI-документация доступна по:

```text
/openapi
```

API принимает и возвращает JSON. Защитите его firewall-ом или private network: API может менять конфигурацию устройства и отправлять directives подключённым станциям.

## Устройства

### `GET /devices`

Возвращает настроенные устройства и статус подключения:

```json
[
  {
    "connected": true,
    "duid": "station-123",
    "name": "Kitchen",
    "quasarConfig": {}
  }
]
```

### `PATCH /devices/:duid`

Обновляет имя и/или конфигурацию Quasar. Поля необязательны:

```json
{
  "name": "Kitchen",
  "quasarConfig": {}
}
```

В ответе возвращаются `duid`, `name` и `quasarConfig`. Quasar отправляет активному соединению directive обновления конфигурации.

### `GET /devices/:duid/directives`

Возвращает имена directives, поддерживаемых подключённым устройством. Каждая запись содержит JSON Schema аргументов и описание.

Если соединения нет, возвращается `404` с кодом `DEVICE_NOT_CONNECTED`.

### `GET /devices/:duid/state`

Возвращает текущее внутреннее состояние подключённого устройства. Если соединения нет, возвращается `404` с кодом `DEVICE_NOT_CONNECTED`.

## Push содержимое

### `POST /devices/:duid/push/text`

Просит устройство произнести текст через настроенный TTS backend. Для всех подключённых устройств используйте `all` вместо `duid`.

```json
{
  "eventText": "Проверка связи"
}
```

При успехе возвращается `{}`.

### `POST /devices/:duid/push/raw`

Отправляет raw Quasar directive. Тело — объект directive, ожидаемый протоколом станции.

```json
{
  "name": "some.directive",
  "payload": {}
}
```

Точная схема directive зависит от протокола подключённого устройства. При создании directives используйте `GET /devices/:duid/directives` и OpenAPI.

### `POST /devices/:duid/push`

Преобразует и отправляет список именованных directives:

```json
[
  {
    "name": "some.directive",
    "arguments": {}
  }
]
```

При успехе возвращается `{}`, а для неподключённого устройства — `404 DEVICE_NOT_CONNECTED`.

## Маршруты function/state

Quasar также предоставляет маршруты интеграции на `FUNCTION_STATE_SERVER_PORT` (по умолчанию 31114 при включённой функции):

- `GET /state` — независимое состояние;
- `PUT /state` — состояние с учётом metadata сессии;
- `GET /functions` — независимые функции;
- `PUT /functions` — функции с учётом metadata сессии;
- `POST /functions` — вызов функции.

Форматы запросов и ответов описаны в [API function/state-сервера](../processor/function-state-server-api.md).

## Ошибки

Маршруты устройства возвращают `DEVICE_NOT_CONNECTED` с HTTP 404, если запрошенная станция не подключена. Неожиданные ошибки возвращаются как `INTERNAL_SERVER_ERROR` с HTTP 500.

Маршруты и схемы определены в `src/main.ts` репозитория [`quasar-server`](https://github.com/v3rt3p/quasar-server). `/openapi` — авторитетный reference для запущенной ревизии.
