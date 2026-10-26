---
layout: default
title: Razvalisa Documentation
---

# Razvalisa

Documentation for deploying and operating the Razvalisa voice assistant stack.

## Deployment

1. Install Docker Engine and Docker Compose v2.
2. Copy `.env.example` to `.env`.
3. Set `OPENAI_API_KEY`, `MQTT_URL`, `POSTGRES_PASSWORD`, and `GLAGOL_JWT_KEY`.
4. Run `docker compose build && docker compose up -d`.
5. Check `GET http://localhost:8090/health` and `GET http://localhost:31121/devices`.

The complete setup is documented in the [deployment walkthrough](deployment.md). The Compose file and `.env` template are stored in this repository as `docker-compose.yml` and `.env.example`.

## Navigation

- [Full deployment walkthrough](deployment.md)
- [Home Assistant and home-functions setup](home-assistant.md)
- [Architecture](architecture.md)
- [Operations](operations.md)
- [Services](services.md)

## Components

- [alice-tts](services/alice-tts.md) - speech synthesis.
- [function-state-server](services/function-state-server.md) - MQTT-backed states and functions.
- [home-functions](services/home-functions.md) - Home Assistant devices as states and functions.
- [https-proxy](services/https-proxy.md) - TLS reverse proxy.
- [nlp-service](services/nlp-service.md) - deterministic parser.
- [processor](services/processor.md) - LLM processor.
- [push-service](services/push-service.md) - MQTT push bridge.
- [quasar-server](services/quasar-server.md) - Quasar API.
- [quasistreaming-gigam](services/quasistreaming-gigam.md) - streaming ASR.
- [speaker-identifier](services/speaker-identifier.md) - speaker embeddings.

## Important

GitHub Pages hosts documentation only. Docker services, secrets, databases, and model workloads must run on your own server or Docker host.
