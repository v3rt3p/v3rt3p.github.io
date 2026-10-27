# Quasar HTTP API

Quasar exposes its administration API on `API_PORT`:

```text
http://quasar-server:31116
```

The interactive OpenAPI document is available at:

```text
/openapi
```

The API accepts and returns JSON. Protect it with a firewall or private network; it can change device configuration and push directives to connected stations.

## Devices

### `GET /devices`

Returns configured devices and whether each one is currently connected.

```json
[
  {
    "connected": true,
    "duid": "station-123",
    "name": "Kitchen",
    "quasarConfig": {}
  }
]
```

### `PATCH /devices/:duid`

Updates the display name and/or Quasar configuration for a device. The fields are optional.

```json
{
  "name": "Kitchen",
  "quasarConfig": {}
}
```

The response contains `duid`, `name`, and `quasarConfig`. Quasar pushes a configuration-update directive to active connections.

### `GET /devices/:duid/directives`

Returns directive names supported by the connected device. Each entry contains an argument JSON Schema and a description.

Returns `404` with `DEVICE_NOT_CONNECTED` when no connection is available.

### `GET /devices/:duid/state`

Returns the current internal state of a connected device. It returns `404` with `DEVICE_NOT_CONNECTED` when the device is not connected.

## Push content

### `POST /devices/:duid/push/text`

Asks the device to speak text through the configured TTS backend. Use `all` as `duid` to target all connected devices.

```json
{
  "eventText": "Проверка связи"
}
```

Returns `{}` on success.

### `POST /devices/:duid/push/raw`

Pushes a raw Quasar directive. The body is the directive object expected by the station protocol.

```json
{
  "name": "some.directive",
  "payload": {}
}
```

The exact directive schema depends on the connected device protocol. Prefer `GET /devices/:duid/directives` and the OpenAPI page when generating directives.

### `POST /devices/:duid/push`

Converts and pushes a list of named directives.

```json
[
  {
    "name": "some.directive",
    "arguments": {}
  }
]
```

Returns `{}` on success and `404 DEVICE_NOT_CONNECTED` if the device is not connected.

## Function/state server routes

Quasar also exposes function/state integration routes on `FUNCTION_STATE_SERVER_PORT` (31114 by default when enabled):

- `GET /state` — independent state;
- `PUT /state` — state depending on session metadata;
- `GET /functions` — independent functions;
- `PUT /functions` — functions depending on session metadata;
- `POST /functions` — execute a function.

The request and response shapes are documented in [Function/state server API](../processor/function-state-server-api.md).

## Errors

Device-specific routes return `DEVICE_NOT_CONNECTED` with HTTP 404 when the requested station is not connected. Unexpected errors are returned as `INTERNAL_SERVER_ERROR` with HTTP 500.
