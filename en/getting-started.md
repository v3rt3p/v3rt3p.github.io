## Requirements

You need:

- Linux
- enough disk space for PostgreSQL and speech models;
- an API key for an OpenAI-compatible LLM;
- DNS control for the Yandex domains used by the proxy;
- a Yandex smart speaker that can be reset and connected to Wi-Fi.

## Setting the host up

First, you'll need to install basic Linux utilities like `wget`, `curl`, `unzip`.

For Debian/Ubuntu you can use the following commands:
```sh
sudo apt update
sudo apt install unzip wget curl
```

For other distros, please either Google, or know how to do this

Then, you'll need to install Docker:

```sh
curl https://get.docker.com | bash
```

## Prepare working directory

Next, you'll need to download STT and TTS assets:

```sh
# STT models
rm -rf quasistreaming-models
mkdir -p quasistreaming-models
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/gigaam_v3_e2e_ctc.onnx -O quasistreaming-models/gigaam_v3_e2e_ctc.onnx
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/gigaam_v3_e2e_ctc_tokens.txt -O quasistreaming-models/gigaam_v3_e2e_ctc_tokens.txt
wget https://v3rt3p-storage.re146.dev/quasistreaming-gigaam/silero_vad.onnx -O quasistreaming-models/silero_vad.onnx

# TTS models
rm -rf supertonic-tts
mkdir -p supertonic-tts
wget https://v3rt3p-storage.re146.dev/supertonic-tts-model.zip -O supertonic-tts/model.zip
unzip supertonic-tts/model.zip -d supertonic-tts/
rm -f supertonic-tts/model.zip

# Base LLM prompts
rm -f prompt.handlebars prompt-state.handlebars
wget https://raw.githubusercontent.com/v3rt3p/processor/refs/heads/master/prompt.handlebars -O prompt.handlebars
wget https://raw.githubusercontent.com/v3rt3p/processor/refs/heads/master/prompt-state.handlebars -O prompt-state.handlebars
wget https://raw.githubusercontent.com/v3rt3p/processor/refs/heads/master/prompt-functions.handlebars -O prompt-functions.handlebars
```

## Prepare external services

As this getting-started setup is simple, we'll need some external LLM API. As an example here we can use OpenRouter's free models, like `nvidia/nemotron-3.5-lightning:free`. For this, you should register on [OpenRouter](https://openrouter.ai).

## Setup Docker compose config

Simplest Docker compose configuration is shown here:
```yaml
services:
  # HTTPS proxy that uses eMudhra certificates
  https-proxy:
    build: https://github.com/v3rt3p/https-proxy.git
    restart: always
    environment:
      YANDEX_UPSTREAM_URL: http://quasar-server:31115 
    ports:
      - 80:80
      - 443:443
  # Main Quasar protocol server, it handles all the 'maind' connections
  quasar-server:
    build: https://github.com/v3rt3p/quasar-server.git
    restart: always
    ports:
      - 31116:31116 # This is an API port for quasar-server, where you can control your smart speakers externally
    environment:
      ALLOW_UPDATES: 'true' # Allow updates to the latest unpatched version
      FUNCTION_STATE_SERVER_PORT: "31114"
      PORT: "31115"
      API_PORT: "31116"
      TTS_OPENAI_BASE_URL: http://supertonic-tts:9150
      TTS_OPENAI_VOICE: F1 # Voice that you will use - in current setup they are: M1-M5 (male) and F1-F6 (female)
      PROCESSOR_BASIC_URL: http://processor:8080/process
      STT_GIGAAM_URL: ws://quasistreaming-gigam:8080
      SQLITE_PATH: /database/database.sqlite3
      TZ: Europe/Moscow # Default timezone for new smart speakers, but you may change through API it after speaker is connected
  volumes:
    - ./quasar-server-db:/database
  # STT service that is based on Sber's GigaAM
  quasistreaming-gigam:
    build:
      context: https://github.com/v3rt3p/quasistreaming-gigam.git
      dockerfile: Dockerfile.cpu # CPU is used by default, if you want more performance, look at the GPU setup guide in services/quasistreaming-gigaam
    restart: always
    environment:
      RECOGNIZER_MODEL_PATH: /models/gigaam_v3_e2e_ctc.onnx
      RECOGNIZER_TOKENS_PATH: /models/gigaam_v3_e2e_ctc_tokens.txt
      VAD_MODEL_PATH: /models/silero_vad.onnx
      # Different VAD (Voice Activity Detection) settings, defaults work, but you may want to tweak them later
      VAD_MIN_SILENCE_DURATION: 1.25
      VAD_THRESHOLD: 0.48
      INPUT_GAIN: 100
    volumes:
      - ./quasistreaming-models/:/models/:ro
  # Simple TTS service that just works
  supertonic-tts:
    build: https://github.com/v3rt3p/supertonic-tts.git
    restart: always
    environment:
      ASSETS_PATH: /assets
    volumes:
      - ./supertonic-tts/assets/:/assets/:ro
  # Main input request processor - it processes all the stuff that you say and responds with answers
  processor:
    build: https://github.com/v3rt3p/processor.git
    restart: always
    environment:
      OPENAI_BASE_URL: https://some-base-url # Replace this with base URL of your LLM provider. For OpenRouter - https://openrouter.ai/api/v1
      OPENAI_API_KEY: sk-.... # Replace this with your LLM provider API key
      OPENAI_MODEL: nvidia/nemotron-3.5-lightning:free # Replace this with your target model name
      PROCESSOR_FUNCTION_SERVER_URLS: http://quasar-server:31114/functions
      PROCESSOR_STATE_SERVER_URLS: http://quasar-server:31114/state
      PROCESSOR_PROMPT_TEMPLATE_PATH: /prompt.handlebars
      PROCESSOR_STATE_PROMPT_TEMPLATE_PATH: /prompt-state.handlebars
      PROCESSOR_FUNCTIONS_PROMPT_TEMPLATE_PATH: /prompt-functions.handlebars
      TZ: Europe/Moscow # Default timezone for new smart speakers, but you may change through API it after speaker is connected
      CACHE_SIZE: 1000
    volumes:
      - ./prompt.handlebars:/prompt.handlebars
      - ./prompt-state.handlebars:/prompt-state.handlebars
      - ./prompt-functions.handlebars:/prompt-functions.handlebars
  # NTP server for devices to synchronize their time to
  ntp-server:
    image: cturra/ntp:latest
    restart: always
    ports:
      - "123:123/udp"
    environment:
      NTP_SERVERS: time1.google.com,time2.google.com,time.cloudflare.com
```

## Configure DNS and the proxy

Point the Yandex domains listed in the current [`https-proxy` domain file](https://github.com/v3rt3p/https-proxy/blob/master/domains/yandex.txt) at the deployment host. 

Also, you MUST point `ntp.yantp.net` at the deployment host for NTP to work (otherwise you will be unable to reset devices or properly use all the functionality)

The proxy generates runtime certificates according to its own configuration.

## Start the whole stack

Start the whole stack by running these commands:

```sh
docker compose up -d --build
```

After some time, all the containers will be up and running.

## Connect a station

Reset the station if it still has an old account or network configuration. Connect it to Wi-Fi using the official [Дом с Алисой](https://play.google.com/store/apps/details?id=com.yandex.iot) application, then watch `quasar-server` and `https-proxy` logs while it connects:

```sh
docker compose logs -f quasar-server https-proxy
```

You should see requests to endpoints like `/register` and `/token` if everything works

## Troubleshooting

- No station traffic: check DNS, ports 80/443, and the proxy upstream.
- No transcript: inspect `quasistreaming-gigam`; check model download, audio format, and CPU memory.
- No answer: inspect `processor`, then verify the LLM URL, key, and model.
- No speech: inspect `supertonic-tts`, its asset directory, and the selected voice (`M1`-`M5` or `F1`-`F5`).
