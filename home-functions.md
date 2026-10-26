# home-functions

`home-functions` — это сервис, который подключается к вашему [Home Assistant](https://www.home-assistant.io/) и превращает устройства с определёнными метками (labels) в функции для Алисы: свет, чайники, вытяжки, телевизоры — всем можно управлять голосом и спрашивать про их состояние и историю использования.

Собран на библиотеке [`@digital-alchemy/hass`](https://github.com/Digital-Alchemy-TS/hass): держит живое websocket-соединение с Home Assistant, так что состояние устройств всегда актуально, а новые устройства подхватываются на лету — без перезапуска.

## Как это работает

- Устройства выбираются **метками из Home Assistant**: повесили метку на объект — он появился у Алисы, сняли — исчез. Списки устройств вести не надо.
- Функции генерируются автоматически из типа устройства и его возможностей (`supported_features`): у лампы будут яркость, цвет и температура, у медиаплеера — громкость и источники, у климата — режимы.
- Домашние имена транслитерируются в префиксы функций: `Люстра` → `lyustra`, `Вытяжка` → `vytyazhka`, `Стеллаж` → `stellazh`.
- У каждого устройства с включением/выключением есть функция `<префикс>_history(days)` — история использования: сколько раз включалось и суммарно сколько работало. «Алиса, сколько раз за неделю кипятился чайник?» — это оно.

Поддерживаются все типы устройств Home Assistant: light, switch, media_player, climate, fan, cover, lock, vacuum, humidifier, water_heater, select, number, scene, script, button, siren, alarm_control_panel, valve, lawn_mower — и generic-фолбэк для всего остального. Read-only домены (`sensor`, `automation`, `update`, ...) не экспортируются — их и размечать не стоит.

## Подготовка в Home Assistant

### 1. Токен долгого доступа

`home-functions` ходит в Home Assistant по долгоживающему токену:

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

И не забудьте указать его процессору — оба списка, функции и состояния:

```yaml
  processor:
    environment:
      PROCESSOR_FUNCTION_SERVER_URLS: http://home-functions:8080
      PROCESSOR_STATE_SERVER_URLS: http://home-functions:8080
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
# Exposed 'fan.tz3000...' as 'vytyazhka' (1 functions)  — по строке на устройство

# список функций (из контейнера processor):
docker compose exec processor wget -q -O- http://home-functions:8080/functions | jq 'keys'
```

Эндпоинты (тот же контракт, что у function-state-server): `GET /functions` (и `POST /functions`), `PATCH /functions` — вызов функции, для функций с ответом возвращает `{ "response": "..." }`, `PUT /state` и `POST /state` — текущие состояния, `GET /health`.

## Что получается у Алисы

Для `light.lampa_1` с именем «Стеллаж» сгенерируются: `set_stellazh_state` (вкл/выкл), `set_stellazh_brightness`, `set_stellazh_color`, `set_stellazh_color_temperature`, `stellazh_history`.

Примеры фраз:

- «Алиса, включи стеллаж» → `set_stellazh_state {state: 1}`
- «Алиса, сделай весь свет фиолетовым на 20 процентов» → цвет + яркость по всем лампам
- «Алиса, сколько раз за неделю включалась вытяжка?» → `vytyazhka_history {days: 7}` — Алиса ответит цифрами
- «Алиса, спокойной ночи» → активируется сцена с такой меткой

Состояния устройств («что с вытяжкой?») подтягиваются из того же живого кэша и попадают в контекст LLM.

## Если что-то не работает

- **Функций 0 / пустой `/functions`** — на объекты не повешена метка из `config.yaml`, или список `labels` пустой. Смотрите логи `Exposed ...`.
- **`401` от Home Assistant** — токен невалиден или отозван, создайте новый.
- **Не коннектится** — проверьте, что `HASS_BASE_URL` доступен **из контейнера** (для host-сети — `http://172.17.0.1:8123`, не `localhost`).
- **У устройства нет истории** — оно было offline весь запрошенный период (история хранит только записанные состояния), либо домен исключён в recorder.
