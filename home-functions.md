# home-functions

**[🇬🇧 English](#english)** · **[🇷🇺 Русский](#русский)**

---

<a name="english"></a>

# home-functions

`home-functions` is a service that connects your [Home Assistant](https://www.home-assistant.io/) to the Alice voice assistant stack and turns devices with specific labels into voice-controllable functions: lights, kettles, cooker hoods, TVs — everything can be controlled by voice, and you can ask about its state and usage history.

It implements the function-state-server HTTP contract, so the processor picks up your devices alongside the other function/state providers.

Built on [`@digital-alchemy/hass`](https://github.com/Digital-Alchemy-TS/hass): it keeps a live websocket connection to Home Assistant, so device states are always current, and newly labeled devices appear on the fly — no restart needed.

## How it works

- Devices are selected by **Home Assistant labels**: put a label on an entity — it appears in Alice, remove it — it disappears. No device lists to maintain.
- Functions are generated automatically from the device domain and its capabilities (`supported_features`): a lamp gets brightness, color and color temperature; a media player gets volume and input sources; a climate unit gets modes.
- Home names are transliterated into function prefixes: `Люстра` → `lyustra`, `Вытяжка` → `vytyazhka`, `Стеллаж` → `stellazh`.
- Every on/off device gets a `<prefix>_history(days)` function — usage history: how many times it was activated and the total active time. "Alice, how many times did the kettle boil this week?" is exactly this function.
- Function arguments are described as JSON Schemas. Functions that return an answer are marked with `hasResponse: true` (history is one of them).
- The current state of every device (`current_<prefix>_state`) is exposed alongside the functions, so "what's up with the hood?" is answered from the same live cache.

All Home Assistant device types are supported: light, switch, media_player, climate, fan, cover, lock, vacuum, humidifier, water_heater, select, number, scene, script, button, siren, alarm_control_panel, valve, lawn_mower — plus a generic on/off fallback for anything else. Read-only domains (`sensor`, `automation`, `update`, ...) are not exported — there is no point in labeling them.

## Preparing Home Assistant

### 1. Long-lived access token

`home-functions` talks to Home Assistant using a long-lived token:

1. Open the Home Assistant web UI → user profile (bottom-left corner).
2. **Security** tab → **Long-lived access tokens**.
3. **Create token**, name it e.g. `home-functions`, copy the value.

The token is shown only once and cannot be created via the API — only through the UI. Never commit it to a repository.

### 2. Label for voice devices

1. **Settings → Labels** (`/config/labels`).
2. **New label**, name it e.g. `Alice`, pick an icon and a color.
3. Remember the label name — it goes into `config.yaml` (the id works too: for the label `Alice` it is `alice`).

### 3. Label your devices

1. **Settings → Devices & services → Entities** (`/config/entities`).
2. Find a device (lamp, fan, TV, ...).
3. Click the entity → enable the **Alice** label in its card.

Good candidates: lights, switches, fans, climate, covers, media players, kettles (water_heater), humidifiers, vacuums, locks, scenes and scripts. Sensors and other read-only things are pointless to label — they are not exported.

A newly labeled device is picked up **without a restart** — the device list is rebuilt from the live registry on every request.

## docker-compose

The service is added to the [getting-started](getting-started.md) stack:

```yaml
  home-functions:
    build: https://github.com/v3rt3p/home-functions.git
    restart: always
    environment:
      HASS_BASE_URL: http://192.168.1.50:8123   # your Home Assistant
      HASS_TOKEN: eyJhbGciOi...                 # token from step 1
    volumes:
      - ./home-functions-config.yaml:/app/config.yaml:ro
    # expose: ["8080"]  # inside the compose network, not published outside
```

And do not forget to point the processor at it — both lists, functions and states. **Note the URL paths**: the processor requests these URLs verbatim, so with the current function-state-server contract they must include the endpoint path:

```yaml
  processor:
    environment:
      PROCESSOR_FUNCTION_SERVER_URLS: http://home-functions:8080/functions
      PROCESSOR_STATE_SERVER_URLS: http://home-functions:8080/state
```

Important: the container must reach Home Assistant over **HTTP and websocket**. If Home Assistant runs in the host network on the same machine as the stack, use the docker bridge IP (`http://172.17.0.1:8123`), not `localhost`.

## Configuration

`home-functions-config.yaml` — at the minimum only the label is required:

```yaml
labels:
  - alice
```

Optionally — rename a device or give modes speakable names:

```yaml
labels:
  - alice

entities:
  light.lampa_1:
    description: Стеллаж              # speakable name, also sets the function prefix
  media_player.tv:
    description: TV
    modes:
      source:                          # which list is being renamed
        HDMI 1: [playstation, приставка]   # [primary name, aliases...]
        HDMI 2: приставка два
      sound_mode:
        Movie: кино
```

Lists that can be renamed: `color`, `effect`, `source`, `sound_mode`, `hvac_mode`, `preset_mode`, `fan_mode`, `swing_mode`, `option`, `operation_mode`, `fan_speed`, `tone`, `mode`. Renaming changes only what the LLM sees — the original value is sent to Home Assistant.

Environment variables:

- `PORT` (default `8080`)
- `CONFIG_PATH` (default `config.yaml`)
- `HASS_BASE_URL` — Home Assistant address
- `HASS_TOKEN` — long-lived access token

## Checking it works

```sh
docker compose up -d --build home-functions
docker compose logs home-functions
# Exposed 'fan.tz3000...' as 'vytyazhka' (2 functions)  — one line per device

# function list (from the processor container):
docker compose exec processor wget -q -O- http://home-functions:8080/functions | jq 'keys'
```

## What Alice can do

For `light.lampa_1` named «Стеллаж» these are generated: `set_stellazh_state` (on/off), `set_stellazh_brightness`, `set_stellazh_color`, `set_stellazh_color_temperature`, `stellazh_history`.

Example phrases:

- "Alice, turn on the shelf light" → `set_stellazh_state {state: 1}`
- "Alice, make all the light purple at 20 percent" → color + brightness across all lamps
- "Alice, how many times was the hood on this week?" → `vytyazhka_history {days: 7}` — Alice answers with numbers
- "Alice, good night" → the scene labeled with that name is activated

## Troubleshooting

- **0 functions / empty `/functions`** — entities are not labeled with the label from `config.yaml`, or the `labels` list is empty. Check the `Exposed ...` logs.
- **`401` from Home Assistant** — the token is invalid or revoked, create a new one.
- **Cannot connect** — check that `HASS_BASE_URL` is reachable **from the container** (for host networking — `http://172.17.0.1:8123`, not `localhost`).
- **No history for a device** — it was offline the whole requested period (history only keeps recorded states), or the domain is excluded in the recorder.
- **Alice says she cannot control devices, but `/functions` is not empty** — the processor cached the old "cannot do that" answers; restart the processor container to flush the cache.

---

<a name="русский"></a>

# home-functions

`home-functions` — это сервис, который подключается к вашему [Home Assistant](https://www.home-assistant.io/) и превращает устройства с определёнными метками (labels) в функции для Алисы: свет, чайники, вытяжки, телевизоры — всем можно управлять голосом и спрашивать про их состояние и историю использования.

Он реализует тот же HTTP-контракт function-state-server, поэтому процессор подхватывает ваши устройства вместе с остальными источниками функций и состояний.

Собран на библиотеке [`@digital-alchemy/hass`](https://github.com/Digital-Alchemy-TS/hass): держит живое websocket-соединение с Home Assistant, так что состояние устройств всегда актуально, а новые устройства подхватываются на лету — без перезапуска.

## Как это работает

- Устройства выбираются **метками из Home Assistant**: повесили метку на объект — он появился у Алисы, сняли — исчез. Списки устройств вести не надо.
- Функции генерируются автоматически из типа устройства и его возможностей (`supported_features`): у лампы будут яркость, цвет и температура, у медиаплеера — громкость и источники, у климата — режимы.
- Домашние имена транслитерируются в префиксы функций: `Люстра` → `lyustra`, `Вытяжка` → `vytyazhka`, `Стеллаж` → `stellazh`.
- У каждого устройства с включением/выключением есть функция `<префикс>_history(days)` — история использования: сколько раз включалось и суммарно сколько работало. «Алиса, сколько раз за неделю кипятился чайник?» — это оно.
- Аргументы функций описываются JSON-схемами. Функции, которые возвращают ответ, помечены `hasResponse: true` (история — одна из них).
- Текущее состояние каждого устройства (`current_<префикс>_state`) экспортируется рядом с функциями, поэтому на «что с вытяжкой?» ответ берётся из того же живого кэша.

Поддерживаются все типы устройств Home Assistant: light, switch, media_player, climate, fan, cover, lock, vacuum, humidifier, water_heater, select, number, scene, script, button, siren, alarm_control_panel, valve, lawn_mower — и generic-фолбэк (вкл/выкл) для всего остального. Read-only домены (`sensor`, `automation`, `update`, ...) не экспортируются — их и размечать не стоит.

## Подготовка в Home Assistant

### 1. Токен долгого доступа

`home-functions` ходит в Home Assistant по долгоживущему токену:

1. Откройте веб-интерфейс Home Assistant → профиль пользователя (левый нижний угол).
2. Вкладка **Безопасность** → **Токены долгого доступа**.
3. **Создать токен**, назовите например `home-functions`, скопируйте значение.

Токен показывается один раз. Через API такой токен создать нельзя — только через интерфейс. Никогда не коммитьте его в репозиторий.

### 2. Метка для голосовых устройств

1. **Настройки → Метки** (`/config/labels`).
2. **Новая метка**, назовите например `Alice`, выберите иконку и цвет.
3. Запомните имя метки — оно пойдёт в `config.yaml` (можно и id: у метки `Alice` это `alice`).

### 3. Разметьте устройства

1. **Настройки → Устройства и службы → Объекты** (`/config/entities`).
2. Найдите устройство (лампа, вентилятор, телевизор, ...).
3. Кликните на объект → в карточке включите метку **Alice**.

Хорошие кандидаты: свет, выключатели, вентиляторы, климат, шторы, медиаплееры, чайники (water_heater), увлажнители, пылесосы, замки, сцены и скрипты. Сенсоры и прочие read-only вещи размечать смысла нет — они не экспортируются.

Новое устройство подхватится **без перезапуска** — список пересобирается на каждый запрос из живого реестра.

## docker-compose

Сервис добавляется к [getting-started](getting-started.md) стеку:

```yaml
  home-functions:
    build: https://github.com/v3rt3p/home-functions.git
    restart: always
    environment:
      HASS_BASE_URL: http://192.168.1.50:8123   # ваш Home Assistant
      HASS_TOKEN: eyJhbGciOi...                 # токен из шага 1
    volumes:
      - ./home-functions-config.yaml:/app/config.yaml:ro
    # expose: ["8080"]  # внутри compose-сети, наружу не публикуется
```

И не забудьте указать его процессору — оба списка, функции и состояния. **Обратите внимание на пути в URL**: процессор запрашивает эти адреса как есть, поэтому по актуальному контракту function-state-server они должны включать путь эндпоинта:

```yaml
  processor:
    environment:
      PROCESSOR_FUNCTION_SERVER_URLS: http://home-functions:8080/functions
      PROCESSOR_STATE_SERVER_URLS: http://home-functions:8080/state
```

Важно: контейнер должен видеть Home Assistant **по HTTP и websocket**. Если Home Assistant работает в host-сети на той же машине, что и стек — используйте IP docker-бриджа (`http://172.17.0.1:8123`), а не `localhost`.

## Конфигурация

`home-functions-config.yaml` — по минимуму нужна только метка:

```yaml
labels:
  - alice
```

Опционально — переименовать устройство или дать режимам говоримые имена:

```yaml
labels:
  - alice

entities:
  light.lampa_1:
    description: Стеллаж              # говорящее имя, задаёт и префикс функций
  media_player.tv:
    description: TV
    modes:
      source:                          # какой список переименовываем
        HDMI 1: [playstation, приставка]   # [основное имя, алиасы...]
        HDMI 2: приставка два
      sound_mode:
        Movie: кино
```

Списки, которые можно переименовывать: `color`, `effect`, `source`, `sound_mode`, `hvac_mode`, `preset_mode`, `fan_mode`, `swing_mode`, `option`, `operation_mode`, `fan_speed`, `tone`, `mode`. Переименование меняет только то, что видит LLM — в Home Assistant уходит исходное значение.

Переменные окружения:

- `PORT` (по умолчанию `8080`)
- `CONFIG_PATH` (по умолчанию `config.yaml`)
- `HASS_BASE_URL` — адрес Home Assistant
- `HASS_TOKEN` — токен долгого доступа

## Проверка

```sh
docker compose up -d --build home-functions
docker compose logs home-functions
# Exposed 'fan.tz3000...' as 'vytyazhka' (2 functions)  — по строке на устройство

# список функций (из контейнера processor):
docker compose exec processor wget -q -O- http://home-functions:8080/functions | jq 'keys'
```

## Что получается у Алисы

Для `light.lampa_1` с именем «Стеллаж» сгенерируются: `set_stellazh_state` (вкл/выкл), `set_stellazh_brightness`, `set_stellazh_color`, `set_stellazh_color_temperature`, `stellazh_history`.

Примеры фраз:

- «Алиса, включи стеллаж» → `set_stellazh_state {state: 1}`
- «Алиса, сделай весь свет фиолетовым на 20 процентов» → цвет + яркость по всем лампам
- «Алиса, сколько раз за неделю включалась вытяжка?» → `vytyazhka_history {days: 7}` — Алиса ответит цифрами
- «Алиса, спокойной ночи» → активируется сцена с такой меткой

## Если что-то не работает

- **Функций 0 / пустой `/functions`** — на объекты не повешена метка из `config.yaml`, или список `labels` пустой. Смотрите логи `Exposed ...`.
- **`401` от Home Assistant** — токен невалиден или отозван, создайте новый.
- **Не коннектится** — проверьте, что `HASS_BASE_URL` доступен **из контейнера** (для host-сети — `http://172.17.0.1:8123`, не `localhost`).
- **У устройства нет истории** — оно было offline весь запрошенный период (история хранит только записанные состояния), либо домен исключён в recorder.
- **Алиса говорит, что не может управлять устройствами, а `/functions` не пустой** — процессор закешировал старые ответы «не могу»; перезапустите контейнер processor, чтобы сбросить кэш.
