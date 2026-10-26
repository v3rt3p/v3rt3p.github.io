# getting-started

Шаги:

0. Установить мелкие тулзы: `unzip`, `wget`, `curl`: например если у вас Ubuntu/Debian, то сделайте так: 
```sh
sudo apt update
sudo apt install unzip wget curl
```
1. Установить `docker`:
```sh
curl https://get.docker.com | bash
```
2. Скачать нужные файлы для моделей и LLM (возможно придется включить VPN - файлы на Cloudflare):
```sh
# Модели для STT
rm -rf quasistreaming-models
mkdir -p quasistreaming-models
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/gigaam_v3_e2e_ctc.onnx -O quasistreaming-models/gigaam_v3_e2e_ctc.onnx
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/gigaam_v3_e2e_ctc_tokens.txt -O quasistreaming-models/gigaam_v3_e2e_ctc_tokens.txt
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/silero_vad.onnx -O quasistreaming-models/silero_vad.onnx

# Модели для TTS
rm -rf supertonic-tts
mkdir -p supertonic-tts
wget https://v3rt3p-storage.re146.dev/supertonic-tts-model.zip -O supertonic-tts/model.zip
unzip supertonic-tts/model.zip -d supertonic-tts/
rm -f supertonic-tts/model.zip

# Стандартные промпты для LLM
rm -f prompt.handlebars prompt-state.handlebars
wget https://raw.githubusercontent.com/v3rt3p/processor/refs/heads/master/prompt.handlebars -O prompt.handlebars
wget https://raw.githubusercontent.com/v3rt3p/processor/refs/heads/master/prompt-state.handlebars -O prompt-state.handlebars
```
3. Иметь какой-нибудь токен к модели - в идеале к чему-то с маленьким Latency. Один из неплохих вариантов - зарегестрироватся на [OpenRouter](https://openrouter.ai) и использовать `nvidia/nemotron-3.5-lightning:free`
4. Создать следующий `docker-compose.yml` и заполнить так, как вам нужно (следуя инструкциям):
```yaml
services:
  https-proxy:
    build: https://github.com/v3rt3p/https-proxy.git
    restart: always
    environment:
      YANDEX_UPSTREAM_URL: http://quasar-server:31115 
    ports:
      - 80:80
      - 443:443
  quasar-server:
    build: https://github.com/v3rt3p/quasar-server.git
    restart: always
    ports:
      - 31116:31116 # Это порт с API quasar-server-а, например что бы отправлять Push-ы или изменять конфиг станции - удобная панелька доступна по /openapi
    environment:
      ALLOW_UPDATES: 'true' # Разрешить ли обновления из ограниченного безопасного (старые сертификаты) списка
      PORT: "31115"
      API_PORT: "31116"
      TTS_OPENAI_BASE_URL: http://supertonic-tts:9150
      TTS_OPENAI_VOICE: F1 # Голос который вы хотите использовать - из текущих вариантов - M1-M5 (male) и F1-F6 (female)
      PROCESSOR_BASIC_URL: http://processor:8080/process
      STT_GIGAAM_URL: ws://quasistreaming-gigam:8080
      POSTGRES_URL: postgres://quasar:quasar@postgres:5432/quasar
  postgres:
    image: postgres:17
    restart: always
    environment:
      POSTGRES_USER: quasar
      POSTGRES_PASSWORD: quasar
      POSTGRES_DB: quasar
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
  quasistreaming-gigam:
    build:
      context: https://github.com/v3rt3p/quasistreaming-gigam.git
      dockerfile: Dockerfile.cpu
    restart: always
    environment:
      RECOGNIZER_MODEL_PATH: /models/gigaam_v3_e2e_ctc.onnx
      RECOGNIZER_TOKENS_PATH: /models/gigaam_v3_e2e_ctc_tokens.txt
      VAD_MODEL_PATH: /models/silero_vad.onnx
      # Разные настройки VAD-а (Voice Activity Detection - обнаружения того что вы перестали говорить, попробуйте разное для себя если что-то не устраивает)
      VAD_MIN_SILENCE_DURATION: 1.25
      VAD_THRESHOLD: 0.48
      INPUT_GAIN: 100
    volumes:
      - ./quasistreaming-models/:/models/:ro
  supertonic-tts:
    build: https://github.com/v3rt3p/supertonic-tts.git
    restart: always
    environment:
      ASSETS_PATH: /assets
    volumes:
      - ./supertonic-tts/assets/:/assets/:ro
  processor:
    build: https://github.com/v3rt3p/processor.git
    restart: always
    environment:
      OPENAI_BASE_URL: https://some-base-url # Заменить на Base URL вашей LLM. Для OpenRouter - https://openrouter.ai/api/v1
      OPENAI_API_KEY: sk-.... # Заменить на API key вашей LLM
      OPENAI_MODEL: nvidia/nemotron-3.5-lightning:free # Собственно модель
      PROCESSOR_FUNCTION_SERVER_URLS: ''
      PROCESSOR_STATE_SERVER_URLS: ''
      PROCESSOR_PROMPT_TEMPLATE_PATH: /prompt.handlebars
      PROCESSOR_STATE_PROMPT_TEMPLATE_PATH: /prompt-state.handlebars
      TZ: Europe/Moscow
      CACHE_SIZE: 1000
    volumes:
      - ./prompt.handlebars:/prompt.handlebars
      - ./prompt-state.handlebars:/prompt-state.handlebars
```      
5. Перенастроить DNS для всех доменов касающихся Яндекса на хост где этот `docker-compose.yml` будет запущен, на данный момент этот список [такой](https://github.com/v3rt3p/https-proxy/blob/master/domains/yandex.txt) (первую строку игнорируем, дальше идут домены, при этом домены которые начинаются с `!` тоже надо добавить как regexp - любой поддомен должен резолвится в наш хост)
6. Запустить `docker-compose.yml`: `docker compose up -d --build`
7. Запустить просмотр логов - может быть полезно: `docker compose logs -f https-proxy`
8. На этом этапе начинается самое интересное - запуск Яндекс.Станции - в идеале в "сброшенном" состоянии - зажав кнопку "Mute" на ней при включении
9. Подключить её к WiFi - для этого можно использовать родное приложение [Дом с Алисой](https://play.google.com/store/apps/details?id=com.yandex.iot)
10. ????
11. PROFIT

Хотите управлять устройствами из Home Assistant голосом — подключите [home-functions](home-functions.md).

Если что-то развалиось - не стесняйтесь, пишите [мне](https://re146.dev)
