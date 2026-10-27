`supertonic-tts` converts Russian text into audio through an OpenAI-compatible speech endpoint. It runs on CPU with ONNX Runtime and loads model assets from a mounted directory.

Source: [`v3rt3p/supertonic-tts`](https://github.com/v3rt3p/supertonic-tts), current `main` branch. This repository has no `master` branch.

## Environment variables

| Variable | Required | Default / example | Why it is used |
| --- | --- | --- | --- |
| `PORT` | No | `9150` | HTTP server port. |
| `ASSETS_PATH` | No | `/app/assets` | Directory containing ONNX models and voice files. |
| `DEFAULT_VOICE` | No | `M1` | Voice used when a request does not choose one. |
| `DEFAULT_SPEED` | No | `1.05` | Default speech speed. |
| `CACHE_SIZE` | No | `256` | Number of generated items kept in memory. |
| `MAX_TEXT_LENGTH` | No | `12000` | Maximum input text length. |

Supported voices are `M1`-`M5` and `F1`-`F5`.

## Compose

```yaml
supertonic-tts:
  build: https://github.com/v3rt3p/supertonic-tts.git
  restart: unless-stopped
  environment:
    ASSETS_PATH: /assets
  volumes:
    - ./supertonic-tts/assets:/assets:ro
```

## API and code

- `GET /health` checks the service.
- `POST /v1/audio/speech` is the preferred OpenAI-compatible endpoint.
- `POST /audio/speech` is also available.