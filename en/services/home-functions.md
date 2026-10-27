`home-functions` connects the stack to Home Assistant: entities that carry a configured label become processor functions and states. Lights, kettles, cooker hoods, TVs — everything can be controlled by voice, and you can ask about its state and usage history.

The service implements the [function/state server API](processor/function-state-server-api.md), so the processor picks up the devices next to the other function and state providers.

It keeps a live WebSocket connection to Home Assistant, so device states are always current and newly labeled devices appear without a restart.

Source: [`v3rt3p/home-functions`](https://github.com/v3rt3p/home-functions), current `main` branch. This repository has no `master` branch.

## Environment variables

| Variable | Required | Default / example | Why it is used |
| --- | --- | --- | --- |
| `HASS_BASE_URL` | Yes | `http://192.168.1.50:8123` | Home Assistant address, reachable from the container over HTTP and WebSocket. |
| `HASS_TOKEN` | Yes | - | Home Assistant long-lived access token. |
| `PORT` | No | `8080` | HTTP server port. |
| `CONFIG_PATH` | No | `config.yaml` | Path to the service configuration file. |

## Compose

```yaml
home-functions:
  build: https://github.com/v3rt3p/home-functions.git
  restart: unless-stopped
  environment:
    HASS_BASE_URL: http://192.168.1.50:8123
    HASS_TOKEN: home-assistant-token
  volumes:
    - ./home-functions/config.yaml:/app/config.yaml:ro
```

Point the processor at the service. Note the URL paths: the processor requests the configured URLs verbatim, so they must include the `/functions` and `/state` endpoint paths:

```yaml
processor:
  environment:
    PROCESSOR_FUNCTION_SERVER_URLS: http://home-functions:8080/functions
    PROCESSOR_STATE_SERVER_URLS: http://home-functions:8080/state
```

If Home Assistant runs in the host network on the same machine, use the Docker bridge IP (`http://172.17.0.1:8123`) instead of `localhost` — inside the container, `localhost` is the container itself.

## Preparing Home Assistant

1. Create a long-lived access token: user profile (bottom-left corner) → **Security** → **Long-lived access tokens** → **Create token**. The token is shown once and can only be created through the UI.
2. Create a label for voice devices: **Settings → Labels** (`/config/labels`). For example `Alice` — its id is `alice`.
3. Label the entities to expose: **Settings → Devices & services → Entities** (`/config/entities`) → open an entity → enable the label.

Good candidates are lights, switches, fans, climate, covers, media players, water heaters, humidifiers, vacuums, locks, scenes and scripts. Read-only domains such as `sensor` are not exported — there is no point in labeling them.

## Configuration

`config.yaml` — only the label list is required:

```yaml
labels:
  - alice
```

Every entity that carries at least one of the listed labels is exposed.

Optional per-entity overrides, keyed by `entity_id`:

```yaml
entities:
  light.lampa_1:
    description: Стеллаж          # display name, also used for the function prefix
    slug: shelf                    # explicit prefix instead of transliteration
  media_player.tv:
    description: TV
    modes:
      source:                      # which list is renamed
        HDMI 1: [playstation, приставка]   # [primary name, aliases...]
        HDMI 2: computer
```

`modes` renames raw Home Assistant list values into speakable names. The key selects the input list: `color`, `effect`, `source`, `sound_mode`, `hvac_mode`, `preset_mode`, `fan_mode`, `swing_mode`, `option`, `operation_mode`, `fan_speed`, `tone`, `mode`. Renaming changes only what the model sees — the original value is sent to Home Assistant.

## How it works

- Devices are selected by labels: attach the label to an entity and it appears in the assistant, remove it and it disappears. No device lists to maintain.
- Functions are generated from the entity domain and its capabilities (`supported_features`): a lamp gets brightness, color and color temperature; a media player gets volume and input sources; a climate unit gets modes.
- Names are transliterated into function prefixes: `Люстра` → `lyustra`, `Вытяжка` → `vytyazhka`, `Стеллаж` → `stellazh`. Set `description` or `slug` to override.
- Every on/off device gets a `<prefix>_history(days)` function — how many times it was activated and the total active time. "How many times did the kettle boil this week?" is exactly this function.
- The current state of every device (`current_<prefix>_state`) is exposed as state with a human-readable summary: brightness, temperature, volume, current track, and so on.

For a light named «Стеллаж» the service generates `set_stellazh_state`, `set_stellazh_brightness`, `set_stellazh_color`, `set_stellazh_color_temperature` and `stellazh_history`.

All Home Assistant device domains are supported: light, switch, media_player, climate, fan, cover, lock, vacuum, humidifier, water_heater, select, number, scene, script, button, siren, alarm_control_panel, valve, lawn_mower — plus a generic on/off fallback for anything else with `turn_on`/`turn_off` services.

## Checking it works

```sh
docker compose up -d --build home-functions
docker compose logs home-functions
# Exposed 'fan.tz3000...' as 'vytyazhka' (2 functions)  — one line per device

# function list, from the processor container:
docker compose exec processor wget -q -O- http://home-functions:8080/functions | jq 'keys'
```

## Troubleshooting

- **No functions / empty `/functions`** — entities do not carry a label from `config.yaml`, or the `labels` list is empty. Check the `Exposed ...` log lines.
- **`401` from Home Assistant** — the token is invalid or revoked; create a new one.
- **Cannot connect** — check that `HASS_BASE_URL` is reachable from the container. For host networking use the Docker bridge IP, not `localhost`.
- **No history for a device** — it was offline for the whole requested period, or the domain is excluded in the recorder.
- **The assistant says it cannot control devices, but `/functions` is not empty** — the processor cached the old "cannot do that" answers; restart the processor container to flush the cache.
