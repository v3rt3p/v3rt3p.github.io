`quasistreaming-gigam` принимает поток PCM-аудио и возвращает промежуточные и финальные русские транскрипции по WebSocket. Текущие версии скачивают checkpoint GigaAM RNNT и используют TEN VAD.

Исходный код: [`v3rt3p/quasistreaming-gigam`](https://github.com/v3rt3p/quasistreaming-gigam), текущая ветка `master`.

## Переменные среды

| Переменная | Обязательна | Значение по умолчанию / пример | Назначение |
| --- | --- | --- | --- |
| `PORT` | Нет | `8080` | WebSocket-порт. |
| `GIGAAM_MODEL_NAME` | Нет | `v3_e2e_rnnt` | Checkpoint GigaAM. |
| `GIGAAM_DEVICE` | Нет | `auto` | Устройство: `cpu`, `cuda` или автоматический выбор. |
| `GIGAAM_DOWNLOAD_ROOT` | Нет | `/app/.cache/gigaam` | Каталог скачивания и кэш моделей. |
| `GIGAAM_MAX_WINDOW_SECONDS` | Нет | `24` | Максимальное окно потокового аудио. |
| `INPUT_GAIN` | Нет | `1` | Усиление входного сигнала. |
| `VAD_THRESHOLD` | Нет | `.3` | Порог определения речи. |
| `VAD_MIN_SILENCE_DURATION` | Нет | `.8` | Длительность тишины для завершения речи. |
| `VAD_MIN_SPEECH_DURATION` | Нет | `1.5` | Минимальная длительность речи. |
| `VAD_MAX_SPEECH_DURATION` | Нет | `8` | Максимальная длительность сегмента речи. |
| `ASR_STREAMING_MODE` | Нет | `streaming` | Режим streaming или quasistreaming. |
| `ASR_QUASISTREAMING_STABLE_REPEATS` | Нет | `3` | Число повторов для стабильного результата. |
| `ASR_TARGET_SR` | Нет | `16000` | Требуемая частота дискретизации. |
| `ASR_LANGUAGE` | Нет | `ru` | Язык распознавания. |
| `LOG_LEVEL` / `LOG_JSON` | Нет | Значения образа | Формат и подробность логов. |
| `SENTRY_DSN` | Нет | Пусто | Необязательный error reporting. |

У сервиса есть дополнительные параметры для partial output, VAD-событий, времени end-of-utterance и небезопасных partial results. Добавляйте их только при необходимости; источником истины является `asr_core/config.py`.

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

Перед настройкой GPU установите совместимые драйвер NVIDIA и Container Toolkit. Версии CUDA-образа и PyTorch должны соответствовать текущему Dockerfile репозитория. До подключения к Quasar проверьте загрузку модели.

## Протокол

Точный handshake, audio frames, transcript events и ошибки соединения описаны в [справочнике ASR WebSocket API](quasistreaming-gigam/websocket-api.md).
