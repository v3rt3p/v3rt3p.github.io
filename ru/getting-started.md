## Требования

Вам понадобятся:

- Linux;
- достаточно места для PostgreSQL и речевых моделей;
- API-ключ OpenAI-совместимой LLM;
- доступ к DNS-доменам Яндекса, которые использует proxy;
- Яндекс.Станция, которую можно сбросить и подключить к Wi-Fi.

## Подготовка хоста

Сначала установите базовые Linux-утилиты: `wget`, `curl`, `unzip`.

В Debian/Ubuntu можно выполнить:

```sh
sudo apt update
sudo apt install unzip wget curl
```

Для других дистрибутивов используйте их документацию или установите эти пакеты привычным для вашей системы способом.

Затем установите Docker:

```sh
curl https://get.docker.com | bash
```

## Подготовка рабочей директории

Скачайте assets для STT и TTS:

```sh
# Модели STT
rm -rf quasistreaming-models
mkdir -p quasistreaming-models
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/gigaam_v3_e2e_ctc.onnx -O quasistreaming-models/gigaam_v3_e2e_ctc.onnx
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/gigaam_v3_e2e_ctc_tokens.txt -O quasistreaming-models/gigaam_v3_e2e_ctc_tokens.txt
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/silero_vad.onnx -O quasistreaming-models/silero_vad.onnx

# Модели TTS
rm -rf supertonic-tts
mkdir -p supertonic-tts
wget https://v3rt3p-storage.re146.dev/supertonic-tts-model.zip -O supertonic-tts/model.zip
unzip supertonic-tts/model.zip -d supertonic-tts/
rm -f supertonic-tts/model.zip

# Базовые промпты LLM
rm -f prompt.handlebars prompt-state.handlebars
wget https://raw.githubusercontent.com/v3rt3p/processor/refs/heads/master/prompt.handlebars -O prompt.handlebars
wget https://raw.githubusercontent.com/v3rt3p/processor/refs/heads/master/prompt-state.handlebars -O prompt-state.handlebars
wget https://raw.githubusercontent.com/v3rt3p/processor/refs/heads/master/prompt-functions.handlebars -O prompt-functions.handlebars
```

## Подготовка внешних сервисов

Поскольку эта конфигурация рассчитана на простой запуск, ей нужна внешняя API LLM. Например, можно использовать бесплатные модели OpenRouter, такие как `nvidia/nemotron-3.5-lightning:free`. Для этого зарегистрируйтесь на [OpenRouter](https://openrouter.ai).

## Настройка Docker Compose

Ниже приведена самая простая конфигурация Docker Compose:

```yaml
services:
  # HTTPS proxy, использующий сертификаты eMudhra
  https-proxy:
    build: https://github.com/v3rt3p/https-proxy.git
    restart: always
    environment:
      YANDEX_UPSTREAM_URL: http://quasar-server:31115
    ports:
      - 80:80
      - 443:443
  # Основной сервер протокола Quasar, обрабатывает все подключения
  quasar-server:
    build: https://github.com/v3rt3p/quasar-server.git
    restart: always
    ports:
      - 31116:31116 # API-порт quasar-server для внешнего управления станциями
    environment:
      ALLOW_UPDATES: 'true' # Разрешить обновление до последней версии без известных патчей
      FUNCTION_STATE_SERVER_PORT: "31114"
      PORT: "31115"
      API_PORT: "31116"
      TTS_OPENAI_BASE_URL: http://supertonic-tts:9150
      TTS_OPENAI_VOICE: F1 # Доступные голоса: M1-M5 и F1-F6
      PROCESSOR_BASIC_URL: http://processor:8080/process
      STT_GIGAAM_URL: ws://quasistreaming-gigam:8080
      POSTGRES_URL: postgres://quasar:quasar@postgres:5432/quasar
  # PostgreSQL хранит конфигурации станций для quasar-server
  postgres:
    image: postgres:17
    restart: always
    environment:
      POSTGRES_USER: quasar
      POSTGRES_PASSWORD: quasar
      POSTGRES_DB: quasar
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
  # STT-сервис на базе GigaAM от Sber
  quasistreaming-gigam:
    build:
      context: https://github.com/v3rt3p/quasistreaming-gigam.git
      dockerfile: Dockerfile.cpu # По умолчанию используется CPU; для GPU см. services/quasistreaming-gigaam
    restart: always
    environment:
      RECOGNIZER_MODEL_PATH: /models/gigaam_v3_e2e_ctc.onnx
      RECOGNIZER_TOKENS_PATH: /models/gigaam_v3_e2e_ctc_tokens.txt
      VAD_MODEL_PATH: /models/silero_vad.onnx
      # Настройки VAD (Voice Activity Detection); значения по умолчанию обычно подходят
      VAD_MIN_SILENCE_DURATION: 1.25
      VAD_THRESHOLD: 0.48
      INPUT_GAIN: 100
    volumes:
      - ./quasistreaming-models/:/models/:ro
  # Простой рабочий TTS-сервис
  supertonic-tts:
    build: https://github.com/v3rt3p/supertonic-tts.git
    restart: always
    environment:
      ASSETS_PATH: /assets
    volumes:
      - ./supertonic-tts/assets/:/assets/:ro
  # Processor входящих запросов: обрабатывает фразы и возвращает ответы
  processor:
    build: https://github.com/v3rt3p/processor.git
    restart: always
    environment:
      OPENAI_BASE_URL: https://some-base-url # Замените на base URL вашего LLM-провайдера; для OpenRouter — https://openrouter.ai/api/v1
      OPENAI_API_KEY: sk-.... # Замените на API-ключ провайдера
      OPENAI_MODEL: nvidia/nemotron-3.5-lightning:free # Замените на имя нужной модели
      PROCESSOR_FUNCTION_SERVER_URLS: http://quasar-server:31114/functions
      PROCESSOR_STATE_SERVER_URLS: http://quasar-server:31114/state
      PROCESSOR_PROMPT_TEMPLATE_PATH: /prompt.handlebars
      PROCESSOR_STATE_PROMPT_TEMPLATE_PATH: /prompt-state.handlebars
      PROCESSOR_FUNCTIONS_PROMPT_TEMPLATE_PATH: /prompt-functions.handlebars
      TZ: Europe/Moscow
      CACHE_SIZE: 1000
    volumes:
      - ./prompt.handlebars:/prompt.handlebars
      - ./prompt-state.handlebars:/prompt-state.handlebars
      - ./prompt-functions.handlebars:/prompt-functions.handlebars
```

## Настройка DNS и proxy

Направьте домены Яндекса из текущего [файла доменов `https-proxy`](https://github.com/v3rt3p/https-proxy/blob/master/domains/yandex.txt) на хост развёртывания.

Proxy генерирует runtime-сертификаты в соответствии со своей конфигурацией.

## Запуск всего стека

Запустите весь стек:

```sh
docker compose up -d --build
```

Через некоторое время все контейнеры должны перейти в состояние running.

## Подключение станции

Сбросьте станцию, если в ней остались старые настройки аккаунта или сети. Подключите её к Wi-Fi через официальное приложение [Дом с Алисой](https://play.google.com/store/apps/details?id=com.yandex.iot), затем смотрите логи `quasar-server` и `https-proxy`:

```sh
docker compose logs -f quasar-server https-proxy
```

При успешном запуске вы должны увидеть запросы к endpoint-ам вроде `/register` и `/token`.

## Если что-то не работает

- Нет трафика станции: проверьте DNS, порты 80/443 и upstream proxy.
- Нет транскрипции: проверьте `quasistreaming-gigam`, скачивание модели, формат аудио и память CPU.
- Нет ответа: проверьте `processor`, URL LLM, ключ и модель.
- Нет речи: проверьте `supertonic-tts`, каталог assets и выбранный голос (`M1`-`M5` или `F1`-`F5`).
