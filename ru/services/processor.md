`processor` превращает распознанный текст и состояние станции в ответ. Он загружает Handlebars-промпты, вызывает OpenAI-совместимую модель и может подключаться к function-, state-, NLP- и MCP-сервисам.

Исходный код: [`v3rt3p/processor`](https://github.com/v3rt3p/processor), текущая ветка `master`.

## Переменные среды

| Переменная | Обязательна | Значение по умолчанию / пример | Назначение |
| --- | --- | --- | --- |
| `PORT` | Нет | `8080` | HTTP- и WebSocket-порт. |
| `OPENAI_API_KEY` | Да | - | API-ключ настроенной LLM; нужен большинству провайдеров. |
| `OPENAI_BASE_URL` | Да | - | Base URL OpenAI-совместимого API. |
| `OPENAI_MODEL` | Да | - | Имя модели, отправляемое провайдеру. |
| `OPENAI_REASONING_EFFORT` | Нет | - | Необязательная настройка reasoning (`high`/`low`/`none` и т. п.). |
| `PROMPT_TEMPLATE_PATH` | Нет | `prompt.handlebars` | Основной prompt. |
| `STATE_PROMPT_TEMPLATE_PATH` | Нет | `prompt-state.handlebars` | Prompt состояния. |
| `FUNCTIONS_PROMPT_TEMPLATE_PATH` | Нет | `prompt-functions.handlebars` | Prompt функций. |
| `PROCESSOR_FUNCTION_SERVER_URLS` | Нет | - | URL function-сервисов через запятую. |
| `PROCESSOR_STATE_SERVER_URLS` | Нет | - | URL state-сервисов через запятую. |
| `PROCESSOR_MCP_SERVER_URLS` | Нет | - | MCP URL в формате `URL#server-name`. |
| `NLP_SERVICE_URL` | Нет | - | Необязательный NLP shortcut-сервис. |
| `NLP_SERVICE_TIMEOUT_MS` | Нет | `500` | Таймаут NLP-запроса. |
| `CACHE_SIZE` | Нет | `1000` | Размер кэша сессий в памяти. |
| `SENTRY_DSN` | Нет | - | Необязательный error reporting. |

`TZ` можно задать в окружении контейнера, но это не основная переменная конфигурации processor.

## Compose

```yaml
processor:
  build: https://github.com/v3rt3p/processor.git
  restart: unless-stopped
  environment:
    OPENAI_BASE_URL: https://llm-base-url
    OPENAI_API_KEY: sk-LLMAPIKEY
    OPENAI_MODEL: llm/model-name
    PROMPT_TEMPLATE_PATH: /prompt.handlebars
    STATE_PROMPT_TEMPLATE_PATH: /prompt-state.handlebars
    FUNCTIONS_PROMPT_TEMPLATE_PATH: /prompt-functions.handlebars
    CACHE_SIZE: "1000"
  volumes:
    - ./prompt.handlebars:/prompt.handlebars:ro
    - ./prompt-state.handlebars:/prompt-state.handlebars:ro
    - ./prompt-functions.handlebars:/prompt-functions.handlebars:ro
```

Скачайте prompt-файлы из репозитория.

## Протокол

- [WebSocket API processor-а](processor/websocket-api.md)
- [API function/state-сервера](processor/function-state-server-api.md)
