# Architecture

The main request flow is: a Yandex Station connects to `quasar-server` over WebSocket; Quasar forwards requests to `processor`; processor uses the LLM, NLP, and remote state/function servers. For audio, Quasar calls the GigaAM ASR and TTS backends. `function-state-server` integrates with MQTT, `home-functions` integrates with Home Assistant, while `push-service` receives MQTT events and sends push requests to the Quasar API.

## External dependencies

- PostgreSQL for `quasar-server`.
- An MQTT broker for state/function and push events.
- A Home Assistant instance for `home-functions`.
- An OpenAI-compatible LLM for `processor`.
- Hugging Face for the first GigaAM and Qwen model downloads.
- NVIDIA runtime only for CUDA ASR.

## Compose service names

The Docker network uses `postgres`, `processor`, `function-state-server`, `home-functions`, `nlp-service`, `quasistreaming-gigam`, `alice-tts`, and `quasar-server`. Do not use `localhost` for one container to reach another.
