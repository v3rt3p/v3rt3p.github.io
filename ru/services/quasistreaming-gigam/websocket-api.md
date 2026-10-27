# WebSocket API ASR

`quasistreaming-gigam` принимает WebSocket на порту `8080`:

```text
ws://quasistreaming-gigam:8080
```

Для конфигурации и событий используются JSON text frames, для аудио — binary frames с PCM16.

## Начало сессии

Первым сообщением должен быть JSON с частотой дискретизации:

```json
{
  "sample_rate": 16000
}
```

Текущая версия принимает `16000` Hz. При другом значении сервер пишет предупреждение о несовпадении и закрывает соединение.

## Передача аудио

После configuration frame отправляйте mono signed 16-bit PCM в binary frames. WAV-заголовок не нужен. Text frames после начальной конфигурации сервис игнорирует.

Пример с Python и `websockets`:

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

В реальном клиенте продолжайте отправлять аудио и одновременно читайте события. Одно соединение может использоваться для нескольких фраз: событие конца фразы сбрасывает streaming-сессию, но не закрывает socket.

## События транскрипции

Фрейм транскрипции имеет вид:

```json
{
  "end_of_utt": false,
  "text": "Какая сегодня",
  "result_type": "partial",
  "reason": null
}
```

| Поле | Значение |
| --- | --- |
| `text` | Распознанный текст. Пустые строки не отправляются. |
| `end_of_utt` | `true`, если сервис обнаружил конец фразы. |
| `result_type` | Классификация результата; может быть `null`. |
| `reason` | Необязательная причина события. |

Финальное событие имеет `end_of_utt: true`:

```json
{
  "end_of_utt": true,
  "text": "Какая сегодня погода?",
  "result_type": "final",
  "reason": "silence"
}
```

Сервер отправляет события только после того, как VAD/ASR pipeline получил непустой текст. Не считайте каждый binary frame границей ответа.

## Ошибки соединения

Сервис закрывает сессию, если начальный JSON неверен, отсутствует `sample_rate` или частота не равна `16000`. Закрытие peer во время streaming записывается в лог и не приводит к финальному decode после закрытия соединения.

Реализация находится в `quasistreaming-server.py`, логика аудио и сессий — в `asr_core/streaming_asr.py`. При добавлении полей синхронизируйте страницу с исходным кодом.
