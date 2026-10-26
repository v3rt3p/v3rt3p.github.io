# home-functions

[Home Assistant](https://www.home-assistant.io/) function and state server for `processor`, built on [`@digital-alchemy/hass`](https://github.com/Digital-Alchemy-TS/hass).

Devices are selected by Home Assistant labels: tag entities with a label, list the label in `config.yaml`, and the service exposes the device automatically with functions generated from its live capabilities (`supported_features`, attribute lists). All Home Assistant device types are supported: light, switch, media player, climate, fan, cover, lock, vacuum, humidifier, water heater, select, number, scene, script, button, siren, alarm control panel, valve, lawn mower, plus a generic on/off fallback for other domains.

## Docker Compose

```bash
cp config.example.yaml config.yaml
# Edit config.yaml: set labels and optional mode renames
docker compose build home-functions
docker compose up -d home-functions
```

Standalone: `docker build -t home-functions . && docker run --rm -p 8082:8080 -v "$PWD/config.yaml:/app/config.yaml:ro" -e HASS_BASE_URL=http://homeassistant:8123 -e HASS_TOKEN=... home-functions`, then `curl http://localhost:8082/health`.

Inside the shared Compose network the container listens on `8080` and is not published to the host.

## Configuration

`config.yaml` selects devices by label; per-entity overrides are optional:

```yaml
labels: [alice]

entities:
  media_player.tv:
    description: TV
    modes:
      source:
        HDMI 1: [playstation, приставка]
```

- `labels` - entities carrying any of these Home Assistant labels are exposed.
- `entities.<entity_id>.description` - override the device name (also used for function prefixes).
- `entities.<entity_id>.slug` - override the transliterated snake_case prefix.
- `entities.<entity_id>.modes` - rename modes/inputs per list (`color`, `effect`, `source`, `sound_mode`, `hvac_mode`, `preset_mode`, `fan_mode`, `swing_mode`, `option`, `operation_mode`, `fan_speed`, `tone`, `mode`). A rename value is a display name or a list of `[primary name, aliases...]`; the raw Home Assistant value is preserved.

Example generated functions for `light.living_room`: `set_living_room_state`, `set_living_room_brightness`, `set_living_room_color`, `set_living_room_color_temperature`, `set_living_room_effect`. The `set_<device>_state` convention (`0=off`, `1=on`) matches the NLP parser and enables power-sound feedback.

## Environment

- `PORT` (default `8080`)
- `CONFIG_PATH` (default `config.yaml`)
- `HASS_BASE_URL` - Home Assistant URL
- `HASS_TOKEN` - long-lived access token

Endpoints: `GET /health`, `GET /functions`, `PATCH /functions`, `POST /state`, and `PUT /state`. The service must be able to reach Home Assistant over HTTP and websocket. Never commit `HASS_TOKEN`.

For the full Home Assistant side (token, labels, device tagging) see the [Home Assistant setup guide](../home-assistant.md).
