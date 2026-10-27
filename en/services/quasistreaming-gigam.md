`quasistreaming-gigam` receives streaming PCM audio and returns partial and final Russian transcripts over WebSocket. Current versions download a GigaAM RNNT checkpoint and use TEN VAD

Source: [`v3rt3p/quasistreaming-gigam`](https://github.com/v3rt3p/quasistreaming-gigam), current `master`.

## Environment variables

| Variable | Required | Default / example | Why it is used |
| --- | --- | --- | --- |
| `PORT` | No | `8080` | WebSocket server port. |
| `GIGAAM_MODEL_NAME` | No | `v3_e2e_rnnt` | GigaAM checkpoint to load. |
| `GIGAAM_DEVICE` | No | `auto` | `cpu`, `cuda`, or automatic device selection. |
| `GIGAAM_DOWNLOAD_ROOT` | No | `/app/.cache/gigaam` | Model download/cache directory. |
| `GIGAAM_MAX_WINDOW_SECONDS` | No | `24` | Maximum streaming audio window. |
| `INPUT_GAIN` | No | `1` | Input gain applied before recognition. |
| `VAD_THRESHOLD` | No | `.3` | Voice activity threshold. |
| `VAD_MIN_SILENCE_DURATION` | No | `.8` | Silence needed to close speech. |
| `VAD_MIN_SPEECH_DURATION` | No | `1.5` | Minimum speech duration. |
| `VAD_MAX_SPEECH_DURATION` | No | `8` | Maximum speech segment duration. |
| `ASR_STREAMING_MODE` | No | `streaming` | Streaming mode (`streaming` or `quasistreaming`). |
| `ASR_QUASISTREAMING_STABLE_REPEATS` | No | `3` | Stability repeats for quasistreaming output. |
| `ASR_TARGET_SR` | No | `16000` | Required target sample rate. |
| `ASR_LANGUAGE` | No | `ru` | Recognition language. |
| `LOG_LEVEL` / `LOG_JSON` | No | Image defaults | Logging format and verbosity. |
| `SENTRY_DSN` | No | Empty | Optional error reporting. |

The service has additional tuning variables for partial output, VAD events, end-of-utterance timing, and unsafe partial results. Add them only when you need that tuning; the repository's `asr_core/config.py` is default.

## CPU Compose

```yaml
quasistreaming-gigam:
  build:
    context: https://github.com/v3rt3p/quasistreaming-gigam.git
    dockerfile: Dockerfile.cpu
  restart: unless-stopped
  environment:
    GIGAAM_DEVICE: cpu
    GIGAAM_DOWNLOAD_ROOT: /app/.cache/gigaam
  volumes:
    - gigaam-cache:/app/.cache/gigaam
```

## CUDA Compose

```yaml
quasistreaming-gigam:
  build:
    context: https://github.com/v3rt3p/quasistreaming-gigam.git
    dockerfile: Dockerfile.cuda
  environment:
    GIGAAM_DEVICE: cuda
    GIGAAM_DOWNLOAD_ROOT: /app/.cache/gigaam
  volumes:
    - gigaam-cache:/app/.cache/gigaam
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
```

Install a compatible NVIDIA driver and Container Toolkit first for GPU setup. The CUDA image and PyTorch version must match the repository's current Dockerfile. Test model loading before putting the service behind Quasar.

## Protocol

See the [ASR WebSocket API reference](quasistreaming-gigam/websocket-api.md) for the handshake, audio frames, transcript events, and connection errors.