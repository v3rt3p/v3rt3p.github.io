# ASR WebSocket API

`quasistreaming-gigam` accepts a WebSocket connection on port `8080`:

```text
ws://quasistreaming-gigam:8080
```

The protocol uses JSON text frames for configuration and transcript events, and binary PCM16 audio frames for audio.

## Start a session

The first frame must be a JSON object containing the sample rate:

```json
{
  "sample_rate": 16000
}
```

The current service accepts `16000` Hz. If the value is different, the server logs a sample-rate mismatch and closes the connection.

## Send audio

After the configuration frame, send mono signed 16-bit PCM samples as binary frames. There is no WAV header. The service ignores text frames after the initial configuration.

Example with Python and `websockets`:

```python
import asyncio
import json
import websockets

async def main():
    async with websockets.connect("ws://localhost:8080") as ws:
        await ws.send(json.dumps({"sample_rate": 16000}))
        with open("audio.pcm", "rb") as audio:
            while chunk := audio.read(3200):
                await ws.send(chunk)
        async for message in ws:
            print(json.loads(message))

asyncio.run(main())
```

In a live client, keep sending audio while reading events. The server can recognize more than one utterance on the same connection; an end-of-utterance event resets the streaming session but does not close the socket.

## Transcript events

Transcript frames have this shape:

```json
{
  "end_of_utt": false,
  "text": "Какая сегодня",
  "result_type": "partial",
  "reason": null
}
```

Fields:

| Field | Meaning |
| --- | --- |
| `text` | Recognized text. Empty text is not emitted. |
| `end_of_utt` | `true` when the service detected an end of utterance. |
| `result_type` | Result classification from the recognizer. It may be `null`. |
| `reason` | Optional reason for the event. |

A final event has `end_of_utt: true`:

```json
{
  "end_of_utt": true,
  "text": "Какая сегодня погода?",
  "result_type": "final",
  "reason": "silence"
}
```

The current server emits transcript events only after its VAD/ASR pipeline has produced non-empty text. Do not treat every binary frame as a response boundary.

## Connection errors

The server closes a session when the initial JSON is invalid, `sample_rate` is missing, or the sample rate is not `16000`. A peer close during streaming is logged and does not cause a final decode after the peer is gone.
