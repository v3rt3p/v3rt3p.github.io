`quasar-server` работает с протоколом устройства, предоставляет API администрирования, связывает ASR/TTS/LLM и хранит постоянную конфигурацию устройства в PostgreSQL.

Исходный код: [`v3rt3p/quasar-server`](https://github.com/v3rt3p/quasar-server), текущая ветка `master`.

## Переменные среды

| Переменная | Обязательна | Значение по умолчанию / пример | Назначение |
| --- | --- | --- | --- |
| `ALLOW_UPDATES` | Нет | `true` | Разрешает обновления до последней версии без известных патчей. |
| `PORT` | Нет | `31115` | Listener протокола станции. |
| `API_PORT` | Нет | `31116` | Listener API администрирования и OpenAPI. |
| `FUNCTION_STATE_SERVER_PORT` | Нет | `31114` | Порт локального function/state-сервера для управления другими устройствами из `processor`. |
| `POSTGRES_URL` | Да | - | Строка подключения PostgreSQL. |
| `PROCESSOR_BASIC_URL` | Да | - | Endpoint processor. |
| `STT_GIGAAM_URL` | Да | - | WebSocket endpoint потокового ASR. |
| `TTS_OPENAI_BASE_URL` | Да | `http://10.0.3.137:8000` | Base URL OpenAI-совместимого TTS. |
| `TTS_OPENAI_API_KEY` | Нет | - | Необязательный ключ TTS backend-а. |
| `TTS_OPENAI_MODEL` | Нет | - | Необязательное имя TTS-модели. |
| `TTS_OPENAI_VOICE` | Нет | - | Голос, передаваемый TTS. |
| `TTS_OPENAI_SPEED` | Нет | `1` | Скорость TTS. |
| `AUDIO_METADATA_URLS` | Нет | - | URL провайдеров audio metadata через запятую. |
| `MUSIC_SERVICE_BASIC_URL` | Нет | - | Необязательный endpoint музыкального сервиса. |
| `GLAGOL_JWT_KEY` | Нет | Случайное значение | Ключ подписи JWT для Glagol API. Для стабильной работы задайте явно. |
| `SENTRY_DSN` | Нет | - | Необязательный error reporting. |
| `UPDATES_CONFIG_PATH` | Нет | Значение образа | Путь к конфигурации обновлений. |

## Compose

```yaml
quasar-server:
  build: https://github.com/v3rt3p/quasar-server.git
  environment:
    PROCESSOR_BASIC_URL: http://processor:8080
    STT_GIGAAM_URL: ws://quasistreaming-gigam:8080
    TTS_OPENAI_BASE_URL: http://supertonic-tts:9150
    TTS_OPENAI_VOICE: F1
    POSTGRES_URL: postgres://quasar:quasar@postgres:5432/quasar
  ports:
    - "31116:31116"
```

## API и код

API слушает `API_PORT`. [Справочник Quasar HTTP API](quasar-server/http-api.md) содержит стабильные маршруты для устройств и push; `/openapi` остаётся авторитетной схемой для запущенной ревизии.
