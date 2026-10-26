# Home Assistant and home-functions setup

This guide covers the Home Assistant side and the `home-functions` side of the integration: creating an access token and labels in Home Assistant, selecting devices, and wiring `home-functions` into the stack.

## Home Assistant side

### 1. Create a long-lived access token

`home-functions` authenticates with a long-lived token:

1. Open Home Assistant web UI → click your user profile (left bottom corner).
2. Go to the **Security** tab → **Long-lived access tokens**.
3. Click **Create Token**, name it `home-functions`, copy the value.

The token is shown once. Store it as `HASS_TOKEN` in the stack `.env` and never commit it.

> Tokens cannot be created over the API with another long-lived token — this step must be done in the UI.

### 2. Create a label for voice-exposed devices

Devices are selected **by label**, not listed one by one:

1. Open **Settings → Labels** (`/config/labels`).
2. Click **New label**, name it e.g. `Alice`, pick an icon and color.
3. Remember the label name — it goes into `config.yaml` (label names, not IDs: `Alice` label has label id `alice`, either works).

### 3. Tag the devices you want to expose

1. Open **Settings → Devices & services → Entities** (`/config/entities`).
2. Filter or find a device (e.g. a lamp, a fan, the TV).
3. Click the entity → in the drawer enable the **Alice** label (gear icon → Labels, or the label toggle depending on version).
4. Repeat for every device that should be voice-controllable.

Useful filters: exclude read-only stuff by simply not labeling it. Domains that never make sense for voice control (`sensor`, `automation`, `update`, `device_tracker`, ...) are excluded by `home-functions` automatically.

Good candidates: lights, switches, fans, climate, covers, media players, water heaters, humidifiers, vacuums, locks, scenes, scripts.

`home-functions` rebuilds the device list on every request from the live websocket registry: tag a new entity and it appears in the next dialog **without a restart**.

## home-functions side

### 4. Stack configuration

`.env` of the deployment repository:

```dotenv
HASS_BASE_URL=http://homeassistant.local:8123   # or http://172.17.0.1:8123 for host-network HA
HASS_TOKEN=eyJhbGciOi...                         # the token from step 1
```

The compose service mounts its config read-only:

```yaml
home-functions:
  build: ./home-functions
  restart: unless-stopped
  environment:
    PORT: 8080
    CONFIG_PATH: /app/config.yaml
    HASS_BASE_URL: ${HASS_BASE_URL}
    HASS_TOKEN: ${HASS_TOKEN}
  volumes:
    - ./home-functions/config.yaml:/app/config.yaml:ro
```

`home-functions` must be able to reach Home Assistant over **both HTTP and websocket** (it keeps a live state cache and reads history over REST).

### 5. Select labels in config.yaml

`home-functions/config.yaml`:

```yaml
labels:
  - alice        # label name or label id from step 2
```

### 6. Optional: rename devices and modes

Russian-friendly names are transliterated automatically (`Люстра` → `lyustra`, `Вытяжка` → `vytyazhka`). To override a name, or to give modes speakable names:

```yaml
labels: [alice]

entities:
  light.lampa_1:
    description: Стеллаж          # spoken/visible name, also drives the function prefix
  media_player.tv:
    description: TV
    modes:
      source:                      # mode list to rename (source, effect, hvac_mode, ...)
        HDMI 1: [playstation, приставка]   # [primary name, aliases...]
        HDMI 2: приставка два
      sound_mode:
        Movie: кино
```

Renaming only changes what the assistant sees — the raw Home Assistant value is sent to services untouched.

### 7. Wire the processor

The processor must include `home-functions` in both lists (already done in the stack compose):

```dotenv
PROCESSOR_FUNCTION_SERVER_URLS=http://function-state-server:8080,http://home-functions:8080
PROCESSOR_STATE_SERVER_URLS=http://function-state-server:8080,http://home-functions:8080
```

### 8. Verify

```bash
docker compose build home-functions && docker compose up -d home-functions

# exposed devices and functions:
docker compose exec processor wget -q -O- http://home-functions:8080/functions | jq 'keys'

# state snapshot:
docker compose exec processor wget -q -O- http://home-functions:8080/state --header='content-type: application/json' --post-data='{}' | jq
```

Logs show one line per device: `Exposed 'fan.xxx' as 'vytyazhka' (2 functions)`.

## What you get

For a labeled device the service generates (examples for a light `light.lampa_1` named `Стеллаж`):

| Function | Purpose |
|---|---|
| `set_stellazh_state` | on/off (`0`/`1`) |
| `set_stellazh_brightness` | brightness 0-100 |
| `set_stellazh_color` | named color variants |
| `set_stellazh_color_temperature` | mireds |
| `stellazh_history` | usage stats: activations count and total active time for the last N days (`hasResponse`) |

Voice examples (Russian):

- «Алиса, включи стеллаж» → `set_stellazh_state {state: 1}`
- «Алиса, сделай стеллаж фиолетовым на 20 процентов» → color + brightness
- «Алиса, сколько раз за неделю включалась вытяжка?» → `vytyazhka_history {days: 7}`, the assistant answers with the numbers
- «Алиса, спокойной ночи» → `scene` labeled `Спокойной ночи` activates

Device states («что с вытяжкой?») come from the same live cache and are fed to the LLM as context.

## Troubleshooting

- **0 devices / empty `/functions`** — no entities carry the configured label, or `config.yaml` labels list is empty. Check `docker compose logs home-functions` for `Exposed ...` lines.
- **`401` from Home Assistant** — invalid or revoked `HASS_TOKEN`; create a new one.
- **Connection failures** — check `HASS_BASE_URL` is reachable **from the container** (for host-network HA use the docker bridge IP like `http://172.17.0.1:8123`, not `localhost`).
- **A device has no history** — the entity was offline/unavailable for the whole period (history holds recorded states only), or the recorder excludes its domain.
