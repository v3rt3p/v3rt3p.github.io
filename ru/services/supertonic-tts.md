`supertonic-tts` превращает русский текст в аудио через OpenAI-совместимый speech endpoint. Он работает на CPU с ONNX Runtime и загружает модели из смонтированного каталога.

Исходный код: [`v3rt3p/supertonic-tts`](https://github.com/v3rt3p/supertonic-tts), текущая ветка `main`. В репозитории нет ветки `master`.

## Переменные среды

| Переменная | Обязательна | Значение по умолчанию / пример | Назначение |
| --- | --- | --- | --- |
| `PORT` | Нет | `9150` | HTTP-порт сервера. |
| `ASSETS_PATH` | Нет | `/app/assets` | Каталог ONNX-моделей и файлов голосов. |
| `DEFAULT_VOICE` | Нет | `M1` | Голос по умолчанию, если запрос не указывает другой. |
| `DEFAULT_SPEED` | Нет | `1.05` | Скорость речи по умолчанию. |
| `CACHE_SIZE` | Нет | `256` | Количество сгенерированных элементов в памяти. |
| `MAX_TEXT_LENGTH` | Нет | `12000` | Максимальная длина входного текста. |

Поддерживаются голоса `M1`-`M5` и `F1`-`F5`.

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

Скачайте модели по README репозитория. В каталоге должны находиться ожидаемые ONNX-файлы и файлы голосов.

## API и код

- `GET /health` проверяет состояние сервиса;
- `POST /v1/audio/speech` — основной OpenAI-совместимый endpoint;
- также доступен `POST /audio/speech`.
