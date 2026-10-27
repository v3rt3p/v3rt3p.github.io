Старая конфигурация состоит из пяти прикладных сервисов и PostgreSQL. Ссылки ниже ведут на текущие репозитории и описание переменных для документированной Compose-конфигурации.

| Сервис | Назначение | Внутренний endpoint |
| --- | --- | --- |
| [`https-proxy`](services/https-proxy.md) | Reverse proxy и TLS-точка входа для Яндекса | `80`, `443` |
| [`quasar-server`](services/quasar-server.md) | Протокол станции, API, оркестрация и хранение данных | `31115`, `31116` |
| [`quasistreaming-gigam`](services/quasistreaming-gigam.md) | Потоковое распознавание речи | WebSocket `8080` |
| [`supertonic-tts`](services/supertonic-tts.md) | Синтез русской речи | HTTP `9150` |
| [`processor`](services/processor.md) | Промпты, вызовы LLM и дополнительные инструменты | WebSocket `/process` на `8080` |
| [`home-functions`](services/home-functions.md) | Устройства Home Assistant как функции и состояния | HTTP `8080` |
| PostgreSQL | Постоянное хранилище данных Quasar | PostgreSQL `5432` |

Сервисы общаются через приватную сеть Docker Compose. Публикуйте только действительно необходимые порты.
