`processor` turns recognized text and station state into an answer. It loads Handlebars prompt templates, calls an OpenAI-compatible model, and can connect to function, state, NLP, and MCP services.

Source: [`v3rt3p/processor`](https://github.com/v3rt3p/processor), current `master`.

## Environment variables

| Variable | Required | Default / example | Why it is used |
| --- | --- | --- | --- |
| `PORT` | No | `8080` | HTTP and WebSocket server port. |
| `OPENAI_API_KEY` | Yes | - | API key for the configured LLM. Required by most providers. |
| `OPENAI_BASE_URL` | Yes | - | OpenAI-compatible API base URL. |
| `OPENAI_MODEL` | Yes | - | Model name sent to the provider. |
| `OPENAI_REASONING_EFFORT` | No | - | Optional provider reasoning setting (`high`/`low`/`none`/etc). |
| `PROMPT_TEMPLATE_PATH` | No | `prompt.handlebars` | Main prompt template. |
| `STATE_PROMPT_TEMPLATE_PATH` | No | `prompt-state.handlebars` | State prompt template. |
| `FUNCTIONS_PROMPT_TEMPLATE_PATH` | No | `prompt-functions.handlebars` | Functions prompt template. |
| `PROCESSOR_FUNCTION_SERVER_URLS` | No | - | Comma-separated function server URLs. |
| `PROCESSOR_STATE_SERVER_URLS` | No | - | Comma-separated state server URLs. |
| `PROCESSOR_MCP_SERVER_URLS` | No | - | MCP URLs in `URL#server-name` form. |
| `NLP_SERVICE_URL` | No | - | Optional NLP shortcut service. |
| `NLP_SERVICE_TIMEOUT_MS` | No | `500` | NLP request timeout. |
| `CACHE_SIZE` | No | `1000` | In-memory session/cache size. |
| `SENTRY_DSN` | No | - | Optional error reporting. |

`TZ` may be set in a container environment, but it is not one of the processor's core configuration values.

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

Download the prompt templates from the repository.

## Protocol

- [Processor WebSocket API](processor/websocket-api.md)
- [Function/state server API](processor/function-state-server-api.md)