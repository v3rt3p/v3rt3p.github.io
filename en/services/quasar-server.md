`quasar-server` speaks the device-facing protocol, exposes the administration API, coordinates ASR/TTS/LLM calls, and stores persistent device configuration in PostgreSQL.

Source: [`v3rt3p/quasar-server`](https://github.com/v3rt3p/quasar-server), current `master`.

## Environment variables

| Variable | Required | Default / example | Why it is used |
| --- | --- | --- | --- |
| `ALLOW_UPDATES` | No | `true` | Allows updates from the configured safe unpatched update list. |
| `PORT` | No | `31115` | Station-facing HTTP/protocol listener. |
| `API_PORT` | No | `31116` | Administration and OpenAPI listener. |
| `FUNCTION_STATE_SERVER_PORT` | No | `31114` | Local function/state server port to use with `processor` to control other devices. |
| `POSTGRES_URL` | Yes | - | PostgreSQL connection string. |
| `PROCESSOR_BASIC_URL` | Yes | - | Processor endpoint used for processing. |
| `STT_GIGAAM_URL` | Yes | - | Streaming ASR WebSocket endpoint. |
| `TTS_OPENAI_BASE_URL` | Yes | `http://10.0.3.137:8000` | OpenAI-compatible TTS base URL. |
| `TTS_OPENAI_API_KEY` | No | - | Optional key passed to the TTS backend. |
| `TTS_OPENAI_MODEL` | No | - | Optional model name for the TTS backend. |
| `TTS_OPENAI_VOICE` | No | - | Voice requested from TTS. |
| `TTS_OPENAI_SPEED` | No | `1` | TTS speed value. |
| `AUDIO_METADATA_URLS` | No | - | Comma-separated audio metadata provider URLs. |
| `MUSIC_SERVICE_BASIC_URL` | No | - | Optional music service endpoint. |
| `GLAGOL_JWT_KEY` | No | Random default | JWT signing key for the Glagol API. Set it explicitly for stable deployments. |
| `SENTRY_DSN` | No | - | Optional error reporting. |
| `UPDATES_CONFIG_PATH` | No | Image default | Path to update configuration. |

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

## API and code

The API listens on `API_PORT`. The [Quasar HTTP API reference](quasar-server/http-api.md) lists the stable device and push routes; `/openapi` remains the authoritative schema for the running revision.
