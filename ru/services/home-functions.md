`home-functions` подключает стек к Home Assistant: объекты с настроенной меткой (label) становятся функциями и состояниями для процессора. Свет, чайники, вытяжки, телевизоры — всем можно управлять голосом и спрашивать про их состояние и историю использования.

Сервис реализует [API function/state-сервера](processor/function-state-server-api.md), поэтому процессор подхватывает устройства вместе с остальными источниками функций и состояний.

Он держит живое WebSocket-соединение с Home Assistant, так что состояние устройств всегда актуально, а новые размеченные устройства подхватываются без перезапуска.

Исходный код: [`v3rt3p/home-functions`](https://github.com/v3rt3p/home-functions), текущая ветка `main`. В репозитории нет ветки `master`.

## Переменные среды

| Переменная | Обязательна | Значение по умолчанию / пример | Назначение |
| --- | --- | --- | --- |
| `HASS_BASE_URL` | Да | `http://192.168.1.50:8123` | Адрес Home Assistant, доступный из контейнера по HTTP и WebSocket. |
| `HASS_TOKEN` | Да | - | Токен долгого доступа Home Assistant. |
| `PORT` | Нет | `8080` | HTTP-порт сервера. |
| `CONFIG_PATH` | Нет | `config.yaml` | Путь к файлу конфигурации сервиса. |

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

Укажите сервис процессору. Обратите внимание на пути в URL: процессор запрашивает эти адреса как есть, поэтому они должны включать пути эндпоинтов `/functions` и `/state`:

```yaml
processor:
  environment:
    PROCESSOR_FUNCTION_SERVER_URLS: http://home-functions:8080/functions
    PROCESSOR_STATE_SERVER_URLS: http://home-functions:8080/state
```

Если Home Assistant работает в host-сети на той же машине, используйте IP Docker-бриджа (`http://172.17.0.1:8123`) вместо `localhost` — внутри контейнера `localhost` указывает на сам контейнер.

## Подготовка Home Assistant

1. Создайте токен долгого доступа: профиль пользователя (левый нижний угол) → **Безопасность** → **Токены долгого доступа** → **Создать токен**. Токен показывается один раз, создать его через API нельзя — только через интерфейс.
2. Создайте метку для голосовых устройств: **Настройки → Метки** (`/config/labels`). Например `Alice` — её id `alice`.
3. Разметьте объекты: **Настройки → Устройства и службы → Объекты** (`/config/entities`) → откройте объект → включите метку.

Хорошие кандидаты: свет, выключатели, вентиляторы, климат, шторы, медиаплееры, чайники (water_heater), увлажнители, пылесосы, замки, сцены и скрипты. Read-only домены вроде `sensor` не экспортируются — размечать их смысла нет.

## Конфигурация

`config.yaml` — обязателен только список меток:

```yaml
labels:
  - alice
```

Экспортируется каждый объект, на котором висит хотя бы одна из перечисленных меток.

Опциональные переопределения по `entity_id`:

```yaml
entities:
  light.lampa_1:
    description: Стеллаж          # отображаемое имя, задаёт и префикс функций
    slug: shelf                    # явный префикс вместо транслитерации
  media_player.tv:
    description: TV
    modes:
      source:                      # какой список переименовываем
        HDMI 1: [playstation, приставка]   # [основное имя, алиасы...]
        HDMI 2: computer
```

`modes` переименовывает «сырые» значения списков Home Assistant в говоримые имена. Ключ выбирает список: `color`, `effect`, `source`, `sound_mode`, `hvac_mode`, `preset_mode`, `fan_mode`, `swing_mode`, `option`, `operation_mode`, `fan_speed`, `tone`, `mode`. Переименование меняет только то, что видит модель — в Home Assistant уходит исходное значение.

## Как это работает

- Устройства выбираются по меткам: повесили метку на объект — он появился у ассистента, сняли — исчез. Списки устройств вести не надо.
- Функции генерируются из домена объекта и его возможностей (`supported_features`): у лампы будут яркость, цвет и температура, у медиаплеера — громкость и источники, у климата — режимы.
- Имена транслитерируются в префиксы функций: `Люстра` → `lyustra`, `Вытяжка` → `vytyazhka`, `Стеллаж` → `stellazh`. Переопределяется через `description` или `slug`.
- У каждого устройства с включением/выключением есть функция `<префикс>_history(days)` — сколько раз включалось и суммарно сколько работало. «Сколько раз за неделю кипятился чайник?» — это оно.
- Текущее состояние каждого устройства (`current_<префикс>_state`) экспортируется как состояние с человекочитаемой сводкой: яркость, температура, громкость, текущий трек и так далее.

Для лампы с именем «Стеллаж» сгенерируются `set_stellazh_state`, `set_stellazh_brightness`, `set_stellazh_color`, `set_stellazh_color_temperature` и `stellazh_history`.

Поддерживаются все домены устройств Home Assistant: light, switch, media_player, climate, fan, cover, lock, vacuum, humidifier, water_heater, select, number, scene, script, button, siren, alarm_control_panel, valve, lawn_mower — и generic-фолбэк (вкл/выкл) для всего остального с сервисами `turn_on`/`turn_off`.

## Проверка

```sh
docker compose up -d --build home-functions
docker compose logs home-functions
# Exposed 'fan.tz3000...' as 'vytyazhka' (2 functions)  — по строке на устройство

# список функций, из контейнера processor:
docker compose exec processor wget -q -O- http://home-functions:8080/functions | jq 'keys'
```

## Если что-то не работает

- **Функций нет / пустой `/functions`** — на объектах не висит метка из `config.yaml`, либо список `labels` пустой. Смотрите строки `Exposed ...` в логах.
- **`401` от Home Assistant** — токен невалиден или отозван; создайте новый.
- **Не коннектится** — проверьте, что `HASS_BASE_URL` доступен из контейнера. Для host-сети используйте IP Docker-бриджа, а не `localhost`.
- **У устройства нет истории** — оно было offline весь запрошенный период, либо домен исключён в recorder.
- **Ассистент говорит, что не может управлять устройствами, а `/functions` не пустой** — процессор закешировал старые ответы «не могу»; перезапустите контейнер processor, чтобы сбросить кэш.
