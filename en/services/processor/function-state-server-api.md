# Function/state server API

Processor integrations use HTTP services for state and functions. Configure their base URLs with `PROCESSOR_STATE_SERVER_URLS` and `PROCESSOR_FUNCTION_SERVER_URLS`.

The shared request and response schemas are defined in [`v3rt3p/types/src/function-state-server.ts`](https://github.com/v3rt3p/types/blob/master/src/function-state-server.ts).

## State endpoints

### `GET /state`

Returns state that does not depend on the current session or device.

```json
{
  "temperature": {
    "description": "Outside temperature",
    "value": "12 °C"
  }
}
```

The response is an object keyed by state name. Each item has a human-readable `description` and `value`.

### `PUT /state`

Returns state that depends on the current session or device.

Request:

```json
{
  "sessionId": "processor-session-id",
  "metadata": {
    "deviceId": "station-123"
  }
}
```

Response:

```json
{
  "room": {
    "description": "Current room",
    "value": "kitchen"
  }
}
```

`metadata` is integration data. `sessionId` identifies the processor WebSocket session and should be treated as opaque.

## Function endpoints

### `GET /functions`

Returns functions that do not depend on the current session or device.

```json
{
  "smart_speaker_tts": {
    "arguments": {
      "type": "object",
      "properties": {
        "text": { "type": "string" }
      },
      "required": ["text"]
    },
    "description": "Say text on the speaker",
    "hasResponse": false
  }
}
```

`arguments` is a JSON Schema describing the function arguments. `hasResponse` tells processor whether it should wait for a result.

### `PUT /functions`

Returns functions that depend on the current session or device.

Request:

```json
{
  "sessionId": "processor-session-id",
  "metadata": {
    "deviceId": "station-123"
  }
}
```

The response has the same shape as `GET /functions`.

### `POST /functions`

Executes a function call.

Request:

```json
{
  "sessionId": "processor-session-id",
  "metadata": {
    "deviceId": "station-123"
  },
  "name": "smart_speaker_tts",
  "arguments": {
    "text": "Включаю свет"
  }
}
```

Response when the function returns data:

```json
{
  "response": {
    "ok": true
  }
}
```

The `response` field is optional. For functions with `hasResponse: false`, return an empty JSON object such as `{}`.

## Implementation notes

Processor first reads independent state/functions with `GET`, then requests dependent values with `PUT` and invokes functions with `POST`. Function servers must accept JSON request bodies and return JSON. Keep the server reachable from the processor container, not only from the host loopback interface.

The Quasar server exposes a compatible function/state server on its internal function-state port. See [Quasar HTTP API](../quasar-server/http-api.md) for its device-facing routes.
