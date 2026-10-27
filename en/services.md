The legacy deployment contains five application services and PostgreSQL. The links below describe the current repositories and the variables used by the documented Compose setup.

| Service | Role | Internal endpoint |
| --- | --- | --- |
| [`https-proxy`](services/https-proxy.md) | Yandex-facing reverse proxy and TLS entry point | `80`, `443` |
| [`quasar-server`](services/quasar-server.md) | Station protocol, API, orchestration, and persistence | `31115`, `31116` |
| [`quasistreaming-gigam`](services/quasistreaming-gigam.md) | Streaming speech recognition | WebSocket `8080` |
| [`supertonic-tts`](services/supertonic-tts.md) | Russian speech synthesis | HTTP `9150` |
| [`processor`](services/processor.md) | Prompting, LLM calls, and optional tools | WebSocket `/process` on `8080` |
| [`home-functions`](services/home-functions.md) | Home Assistant devices as functions and states | HTTP `8080` |
| PostgreSQL | Persistent Quasar data | PostgreSQL `5432` |

These services communicate over the private Docker Compose network. Publish only the ports you actually need.
