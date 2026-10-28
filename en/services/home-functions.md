`home-functions` connects the stack to Home Assistant: entities that carry a configured label become processor functions and states. Available commands depend on the entity domain and capabilities.

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

Add the service to the existing `services` section of your `docker-compose.yml`:

```yaml
services:
  home-functions:
    build: https://github.com/v3rt3p/home-functions.git#main
    restart: unless-stopped
    environment:
      PORT: "8080"
      CONFIG_PATH: /app/config.yaml
      HASS_BASE_URL: ${HASS_BASE_URL:?Set HASS_BASE_URL in .env}
      HASS_TOKEN: ${HASS_TOKEN:?Set HASS_TOKEN in .env}
    volumes:
      - ./home-functions/config.yaml:/app/config.yaml:ro
```

Set `HASS_BASE_URL` and `HASS_TOKEN` in `.env` next to the Compose file. Before starting, create `./home-functions/config.yaml` using the Configuration example; otherwise Docker may create a directory at that path. If the source is already in `./home-functions`, use `build: ./home-functions` instead of the Git URL.

Add these variables to the existing processor service's `environment`, preserving its other settings. The processor requests the configured URLs verbatim, so they must include the `/functions` and `/state` endpoint paths:

```yaml
services:
  processor:
    environment:
      PROCESSOR_FUNCTION_SERVER_URLS: http://home-functions:8080/functions
      PROCESSOR_STATE_SERVER_URLS: http://home-functions:8080/state
```

If these variables already list other servers, append the home-functions URL separated by a comma without spaces. Adding variables to `.env` alone is insufficient: Compose must pass them through the processor's `environment`. Both services need a shared Docker network; publishing the home-functions port on the host is unnecessary.

If Home Assistant runs on the same host, use a host LAN address or Docker gateway reachable from the container. `http://172.17.0.1:8123` only works if that is the gateway address and HA is accessible on that interface. Inside a container, `localhost` refers to the container itself.

## Preparing Home Assistant

1. Create a long-lived access token: user profile → **Security** → **Long-lived access tokens** → **Create token**. Store the token in `HASS_TOKEN`.
2. Create a label for voice devices in Home Assistant (`/config/labels`). The configuration's `labels` list requires its **ID**, not its display name; a new label named `Alice` usually has the ID `alice`.
3. Label the entities to expose: **Settings → Devices & services → Entities** (`/config/entities`) → open an entity → enable the label.

Good candidates are lights, switches, fans, climate, covers, media players, water heaters, humidifiers, vacuums, locks, scenes and scripts. Read-only domains such as `sensor` are not exported — there is no point in labeling them.

## Configuration

`config.yaml` — only the label list is required:

```yaml
labels:
  - alice
```

Supported entities carrying at least one listed label are exposed. Assign the label to the entity itself: this service does not inherit labels from devices or areas. An empty `labels` list exports nothing.

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

Use `description` or the HA entity name for a recognizable voice name. The current implementation does not include the `names` field as aliases in published functions. Function prefixes must be unique; set distinct `slug` values for entities with identical names.

## How it works

- Devices are selected by labels: attach the label to an entity and it appears in the assistant, remove it and it disappears. No device lists to maintain.
- Functions are generated from the entity domain and its capabilities (`supported_features`): a lamp gets brightness, color and color temperature; a media player gets volume and input sources; a climate unit gets modes.
- Names are transliterated into function prefixes: `Люстра` → `lyustra`, `Вытяжка` → `vytyazhka`, `Стеллаж` → `stellazh`. Set `description` or `slug` to override.
- Every on/off device gets a `<prefix>_history(days)` function — how many times it was activated and the total active time. "How many times did the kettle boil this week?" is exactly this function.
- The current state of every device (`current_<prefix>_state`) is exposed as state with a human-readable summary: brightness, temperature, volume, current track, and so on.

For a light named «Стеллаж» the service generates `set_stellazh_state`, `set_stellazh_brightness` and `stellazh_history`. Color and color-temperature functions are added when the corresponding modes appear in `supported_color_modes`.

The processor sends the function catalog and states to the LLM. With a configured model, commands such as "включи Стеллаж" do not require adding phrases to the deterministic parser. `nlp-service` is an optional shortcut; unrecognized commands fall through to the LLM. When several lights exist, name the intended one explicitly: this service does not associate entities with the speaker's room.

Current processor limitation: `RemoteFunctionServer.callFunction` does not read HTTP response bodies. Although home-functions publishes history functions and returns their results, the processor does not yet pass these results to the model. Label configuration cannot fix this.

All Home Assistant device domains are supported: light, switch, media_player, climate, fan, cover, lock, vacuum, humidifier, water_heater, select, number, scene, script, button, siren, alarm_control_panel, valve, lawn_mower — plus a generic on/off fallback for anything else with `turn_on`/`turn_off` services.

## Checking it works

```sh
docker compose config --quiet
docker compose up -d --build home-functions
docker compose up -d processor
docker compose logs home-functions
# Exposed 'fan.tz3000...' as 'vytyazhka' (2 functions)  — one line per device

# function list, from the processor container:
docker compose exec -T processor node -e 'fetch("http://home-functions:8080/functions").then(async r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); console.log(Object.keys(await r.json())) }).catch(e => { console.error(e); process.exit(1) })'
```

`up -d processor` applies environment changes, recreating the container if needed. A plain `restart` does not apply new Compose environment settings. After checking the catalog, speak a command with the exact device name and inspect `docker compose logs --since 5m processor home-functions`: expect the selected function in processor and `Called function` in home-functions. Also check the actual state in HA: a call log alone does not confirm success.

## Troubleshooting

- **No functions / empty `/functions`** — entities do not carry a label from `config.yaml`, or the `labels` list is empty. Check the `Exposed ...` log lines.
- **`401` from Home Assistant** — the token is invalid or revoked; create a new one.
- **Cannot connect** — check that `HASS_BASE_URL` is reachable from the container, HA listens on that interface, and network settings permit the connection.
- **No history for a device** — check HA recorder settings and retention; if home-functions returns history but the model receives none, see the processor limitation above.
- **The assistant says it cannot control devices, but `/functions` is not empty** — check processor environment URLs, catalog-loading errors, and LLM settings. An old cached answer is another possibility: `docker compose restart processor` clears the cache.
- **The speaker says "done" but the device does not change** — check home-functions logs for errors and the entity's availability in HA. The current processor HTTP client does not check function-call response status, so a spoken acknowledgment does not guarantee success.
