# Full Deployment Walkthrough

## 1. What is included

Razvalisa is a set of processes running in one Docker Compose network:

1. `quasar-server` accepts Yandex Station connections and exposes the push API.
2. `processor` accepts WebSocket sessions from Quasar, builds context, and calls an OpenAI-compatible LLM.
3. `nlp-service` handles common commands quickly without an LLM call.
4. `function-state-server` provides states and functions to processor through MQTT-backed providers.
5. `home-functions` exposes Home Assistant devices as states and functions.
6. `quasistreaming-gigam` recognizes incoming audio over WebSocket.
7. `alice-tts` converts text to OGG through Yandex UniProxy.
8. `push-service` forwards MQTT events to the Quasar HTTP API.
9. `speaker-identifier` stores and compares voice embeddings.
10. `postgres` stores Quasar station data.
11. `https-proxy` is an optional external TLS reverse proxy.

GitHub Pages hosts documentation only. Docker containers must run on your server, VPS, or local Docker host.

## 2. Requirements

- Docker Engine 24 or newer.
- Docker Compose v2 (`docker compose version`).
- Git.
- NVIDIA driver, NVIDIA Container Toolkit, and compatible CUDA hardware for GPU ASR.
- Outbound HTTPS for npm/pip packages and GigaAM/Qwen model downloads.
- An accessible MQTT broker.
- An accessible OpenAI-compatible LLM endpoint.

Verify the installation:

```bash
docker --version
docker compose version
git --version
```

## 3. Clone the repositories

The shared `docker-compose.yml` assumes the service repositories are siblings of the `docs` directory:

```bash
mkdir razvalisa && cd razvalisa
git clone https://github.com/v3rt3p/docs.git docs
git clone https://github.com/v3rt3p/alice-tts.git alice-tts
git clone https://github.com/v3rt3p/function-state-server.git function-state-server
git clone https://github.com/v3rt3p/home-functions.git home-functions
git clone https://github.com/v3rt3p/https-proxy.git https-proxy
git clone https://github.com/v3rt3p/nlp-service.git nlp-service
git clone https://github.com/v3rt3p/processor.git processor
git clone https://github.com/v3rt3p/push-service.git push-service
git clone https://github.com/v3rt3p/quasar-server.git quasar-server
git clone https://github.com/v3rt3p/quasistreaming-gigam.git quasistreaming-gigam
git clone https://github.com/v3rt3p/speaker-identifier.git speaker-identifier
```

The remaining commands run from `razvalisa/docs`, where the Compose file is stored.

## 4. Secrets and configuration

```bash
cp .env.example .env
chmod 600 .env
```

Edit `.env` and replace all placeholder values:

```dotenv
POSTGRES_PASSWORD=<long-random-password>
OPENAI_API_KEY=<llm-api-key>
MQTT_URL=mqtts://<user>:<password>@<broker>:8883
GLAGOL_JWT_KEY=<long-random-secret>
HASS_BASE_URL=http://<home-assistant-host>:8123
HASS_TOKEN=<long-lived-home-assistant-token>
```

Never commit `.env`. Do not publish MQTT credentials, LLM keys, `GLAGOL_JWT_KEY`, `ca.key`, or enrollment profiles.

### State and function configuration

Review `function-state-server/config.yaml`. An empty file starts successfully but declares no functions or states. Use `config.example.yaml` as a starting point, then configure the real MQTT topics and external HTTP endpoints.

Review `home-functions/config.yaml`. In Home Assistant, create a label (for example `alice`), assign it to the devices to expose, then list it under `labels:` in the config. Per-entity `modes:` renames are optional; see `home-functions/config.example.yaml`. The service needs `HASS_BASE_URL` and `HASS_TOKEN` to reach Home Assistant. Step-by-step instructions (token, labels, mode renames, verification) are in the [Home Assistant setup guide](home-assistant.md).

### Quasar configuration

The Compose file uses these internal Docker names:

- PostgreSQL: `postgres:5432`.
- Processor: `processor:8080`.
- ASR: `quasistreaming-gigam:8080`.
- TTS: `alice-tts:9150`.
- Quasar API: `quasar-server:31121`.

Do not use `localhost` for one container to contact another.

## 5. First startup

Validate the rendered configuration without starting containers:

```bash
docker compose --env-file .env config
```

Build the images:

```bash
docker compose --env-file .env build
```

Start the stack:

```bash
docker compose --env-file .env up -d
```

Inspect the services:

```bash
docker compose ps
docker compose logs -f quasar-server processor
```

The first ASR and speaker-identifier startup may take a while because models are downloaded from external sources. The model caches are stored in named volumes.

## 6. Verify the deployment

NLP has a formal health endpoint:

```bash
curl -fsS http://localhost:8090/health
```

Check the Quasar API:

```bash
curl -i http://localhost:31121/devices
```

Processor is verified through its `/process` WebSocket endpoint; it has no separate HTTP `/health` endpoint. Check its logs and the Quasar connection.

Test ASR with the client from its repository:

```bash
cd ../quasistreaming-gigam
python client.py input.wav ws://localhost:8080
```

The WAV file must be PCM at 16 kHz. Inside Compose, ASR is available at `ws://quasistreaming-gigam:8080` and is not published to the host by default.

TTS can be tested through Quasar or directly. To test it from the host, temporarily publish port `9150` using a Compose override, then run:

```bash
curl -X POST http://localhost:9150/v1/audio/speech \
  -H 'content-type: application/json' \
  -d '{"input":"Hello"}' \
  --output answer.ogg
```

## 7. GPU ASR

CPU is the default and safest option. For CUDA:

1. Install NVIDIA Container Toolkit.
2. Verify `docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi`.
3. Build `quasistreaming-gigam` with `Dockerfile.cuda`.
4. Set `GIGAAM_DEVICE=cuda`.
5. Add the GPU reservation from `quasistreaming-gigam/docker-compose.yaml`.
6. Confirm that startup logs report `device=cuda`.

If the model falls back to CPU, check the driver, toolkit, PyTorch wheel, and GPU architecture.

## 8. HTTPS proxy

Start the proxy only when required:

```bash
docker compose --env-file .env --profile proxy up -d https-proxy
```

Ports:

- HTTP: `80`.
- HTTPS: `443`.

Replace the CA materials in `https-proxy/certs/` before production. Never publish the private key to GitHub. Configure upstreams with `YANDEX_UPSTREAM_URL`, `VK_UPSTREAM_URL`, and `SBER_UPSTREAM_URL`.

## 9. Data and backups

Compose creates these volumes:

- `postgres-data` - PostgreSQL data.
- `gigaam-cache` - GigaAM checkpoint.
- `speaker-models` - Qwen model cache.
- `speaker-data` - enrollment profiles.

Back up PostgreSQL and the profile directory before upgrades. Do not run `docker compose down -v` unless you intentionally want to delete persistent data.

## 10. Updates and shutdown

```bash
git -C ../alice-tts pull
git -C ../processor pull
git -C ../quasar-server pull
docker compose --env-file .env build
docker compose --env-file .env up -d
```

Stop containers without deleting volumes:

```bash
docker compose down
```

View logs for one service:

```bash
docker compose logs --tail=200 -f SERVICE_NAME
```

## 11. Common problems

- **Empty `function-state-server`:** populate `config.yaml`, otherwise processor sees no functions.
- **`home-functions` exposes nothing:** assign the configured label to entities in Home Assistant and check `HASS_BASE_URL`/`HASS_TOKEN`.
- **Processor cannot call the LLM:** check `OPENAI_API_KEY`, `OPENAI_BASE_URL`, and `OPENAI_MODEL`.
- **Quasar cannot reach processor:** use `http://processor:8080` inside Compose, not `localhost`.
- **ASR takes a long time to start:** wait for the model download and verify the `gigaam-cache` volume.
- **MQTT connection fails:** check the URL, credentials, CA certificate, and network access.
- **PostgreSQL is unavailable:** wait for `service_healthy` and inspect `docker compose logs postgres`.
- **Push events are lost:** `push-service` has no guaranteed delivery or persistent queue; inspect MQTT and API logs.

## 12. Production checklist

- [ ] All secrets are replaced and absent from Git.
- [ ] MQTT uses TLS and ACLs.
- [ ] API ports are protected by a firewall or reverse proxy.
- [ ] PostgreSQL is backed up regularly.
- [ ] `GLAGOL_JWT_KEY` remains stable across restarts.
- [ ] The ASR model is prewarmed.
- [ ] Speaker verification is calibrated with target and impostor recordings.
- [ ] A PostgreSQL migration strategy has been reviewed; do not blindly rely on `synchronize: true`.
- [ ] Restart policy, log monitoring, and disk monitoring are configured.
