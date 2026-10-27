# Processor WebSocket API

The processor accepts one WebSocket connection per conversation at:

```text
ws://processor:8080/process
```

The connection is upgraded only for the `/process` path. Messages are JSON text frames. The shared schemas are defined in [`v3rt3p/types/src/processor.ts`](https://github.com/v3rt3p/types/blob/master/src/processor.ts).

## Message order

1. Open `/process`.
2. Send one `prepare` message with the directive functions available to the current device.
3. Send one or more `process` messages for user input.
4. Read `partialResponse` messages until `finished` is `true`.
5. If `shouldListen` is `true`, keep the conversation open and send the next `process` message. Otherwise the server closes the WebSocket after the response.

## Client messages

### `prepare`

Loads the base prompt, independent state, and functions for the session.

```json
{
  "type": "prepare",
  "data": {
    "directiveFunctions": {
      "smart_speaker_tts": {
        "arguments": {
          "type": "object",
          "properties": {
            "text": { "type": "string" }
          }
        },
        "description": "Speak text on the current speaker"
      }
    }
  }
}
```

`directiveFunctions` is a map from function name to its JSON Schema and description. The processor adds these functions to the prompt; it does not execute them itself unless they are connected through a configured function or MCP server.

### `process`

Processes one recognized utterance.

```json
{
  "type": "process",
  "data": {
    "text": "Какая сегодня погода?",
    "metadata": {
      "internal": {
        "deviceId": "station-123"
      },
      "readyToUse": {
        "room": {
          "description": "Current room",
          "value": "kitchen"
        }
      }
    }
  }
}
```

- `text` is the recognized user text.
- `metadata.internal` is passed to function/state integrations and should not be shown as user-facing state.
- `metadata.readyToUse` contains state already formatted for the LLM.

## Server messages

### `partialResponse`

The processor sends a response after each LLM or tool step.

```json
{
  "type": "partialResponse",
  "data": {
    "directives": [],
    "text": "Сегодня солнечно.",
    "finished": true,
    "shouldListen": false
  }
}
```

Fields:

| Field | Meaning |
| --- | --- |
| `text` | Text to speak or display. |
| `directives` | Device directives returned by the model. Each item has `name` and `arguments`. |
| `finished` | `false` while more processing or tool calls are pending; `true` when this request is complete. |
| `shouldListen` | Present when `finished` is `true`; requests another user utterance when `true`. |

When the model calls a function that expects a response, the processor sends `finished: false`, executes the function, and sends another `partialResponse` after the result is available. A no-response function can still appear in `directives`, but it does not keep the request open.

## Errors and disconnects

Invalid JSON or a message that does not match the shared schema is rejected by the processor's message handler. The service logs the WebSocket error and closes the session. Treat a close before `finished: true` as a failed request and retry only if the station can safely repeat the utterance.