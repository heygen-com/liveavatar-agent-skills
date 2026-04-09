# LITE Mode Implementation Guide

You bring your own AI stack (STT + LLM + TTS). LiveAvatar only renders video from your audio. 1 credit/minute.

```
User audio --> [Your stack: ASR -> LLM -> TTS] --> audio --> [LiveAvatar: Video] --> Avatar stream
```

**LITE Mode was formerly called "Custom Mode."**

## Prerequisites

1. **API key** — https://app.liveavatar.com
2. **Avatar ID** — dashboard or `GET /v1/avatars`
3. **Your TTS must output PCM 16-bit, 24KHz** — wrong format = garbled avatar with NO error

## Session Lifecycle

### Step 1: Create session token (BACKEND, `X-API-KEY`)

```bash
curl -X POST https://api.liveavatar.com/v1/sessions/token \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"mode": "LITE", "avatar_id": "<avatar_id>"}'
```

Returns: `{ "data": { "session_id": "...", "session_token": "..." } }`

### Step 2: Start session (BACKEND, `Bearer <session_token>`)

```bash
curl -X POST https://api.liveavatar.com/v1/sessions/start \
  -H "Authorization: Bearer <session_token>"
```

Returns three things:
```json
{
  "data": {
    "livekit_url": "wss://...",
    "livekit_client_token": "...",
    "ws_url": "wss://..."
  }
}
```

- `livekit_url` + `livekit_client_token` → **frontend** (video stream)
- `ws_url` → **your backend/agent** (audio commands via WebSocket)

### Step 3: Connect both

**Frontend** — join LiveKit room for video using the Web SDK:
```bash
npm install @heygen/liveavatar-web-sdk
```

The SDK handles LiveKit room connection internally — create `new LiveAvatarSession(sessionToken)`, listen for `SESSION_STREAM_READY`, then call `session.attach(videoElement)`. For LITE Mode, the frontend is video-only — your backend handles all audio via WebSocket.

**For frontend implementation details, clone or read the Web SDK repo:**
https://github.com/heygen-com/liveavatar-web-sdk

The `apps/demo/` directory has a complete Next.js example including LITE Mode.

Or quick test: `https://meet.livekit.io/custom?liveKitUrl=<livekit_url>&token=<livekit_client_token>`

**Backend/Agent** — connect WebSocket to `ws_url` for audio commands.

## Event System: WebSocket

**LITE uses a WebSocket, NOT LiveKit data channels. Completely different from FULL Mode.**

### CRITICAL: Wait for "connected"

After connecting to `ws_url`, wait for:
```json
{"type": "session.state_updated", "state": "connected"}
```

**Events sent before `connected` are silently dropped. No error.**

### Commands you send

| Event | Format |
|-------|--------|
| `agent.speak` | `{"type": "agent.speak", "event_id": "<id>", "audio": "<base64-pcm>"}` |
| `agent.speak_end` | `{"type": "agent.speak_end", "event_id": "<id>"}` |
| `agent.interrupt` | `{"type": "agent.interrupt"}` |
| `agent.start_listening` | `{"type": "agent.start_listening", "event_id": "<id>"}` |
| `agent.stop_listening` | `{"type": "agent.stop_listening", "event_id": "<id>"}` |
| `session.keep_alive` | `{"type": "session.keep_alive", "event_id": "<id>"}` |

**`agent.speak_end` is required** after sending audio. Without it, the avatar won't transition to idle/listening.

**`agent.start_listening` / `agent.stop_listening`** control the avatar's visual listening state. When listening is active, the avatar appears attentive — nodding, maintaining eye contact, showing engaged body language. When stopped, the avatar returns to a neutral idle pose. Use `start_listening` when the user begins speaking and `stop_listening` when they finish, so the avatar reacts naturally to conversation turns.

### Events you receive

| Event | Payload |
|-------|---------|
| `session.state_updated` | `{"state": "connected" \| "connecting" \| "closed" \| "closing"}` |
| `agent.speak_started` | `{"event_id": "...", "task": {"id": "..."}}` |
| `agent.speak_ended` | `{"event_id": "...", "task": {"id": "..."}}` |

## Conversation Turn Orchestration

For conversational use cases, follow this turn cycle to keep the avatar's visual state in sync with the conversation:

```
┌─────────────────────────────────────────────────────────┐
│  1. User starts speaking                                │
│     → send agent.start_listening                        │
│     → avatar becomes attentive (nods, eye contact)      │
│                                                         │
│  2. User finishes speaking                              │
│     → send agent.stop_listening                         │
│     → run your pipeline: STT → LLM → TTS               │
│                                                         │
│  3. Stream TTS audio                                    │
│     → send agent.speak chunks (same event_id)           │
│     → send agent.speak_end when stream finishes         │
│     → avatar lip-syncs to audio                         │
│                                                         │
│  4. Avatar finishes speaking                            │
│     → receive agent.speak_ended from server             │
│     → send agent.start_listening                        │
│     → ready for next turn                               │
│                                                         │
│  ✕  User interrupts mid-speech                          │
│     → stop your send loop                               │
│     → send agent.interrupt                              │
│     → send agent.start_listening                        │
│     → go to step 2 when user finishes                   │
└─────────────────────────────────────────────────────────┘
```

```python
# Pseudocode — full conversation loop
while session_active:
    start_listening(ws)
    user_audio = wait_for_user_speech()      # Your VAD / STT
    stop_listening(ws)

    text = transcribe(user_audio)            # Your STT
    response = llm_generate(text)            # Your LLM
    tts_stream = tts_synthesize(response)    # Your TTS (PCM 24KHz)

    stream_audio(ws, tts_stream)             # See "Streaming TTS audio" below
    wait_for_event(ws, "agent.speak_ended")  # Avatar done speaking
```

**This is the recommended pattern for any conversational LITE integration.** The key is always bookending the user's turn with `start_listening` / `stop_listening` so the avatar visually reacts, and always waiting for `agent.speak_ended` before starting the next listening cycle.

## Audio Format

**This is where most LITE integrations break.**

| Parameter | Required value |
|-----------|---------------|
| Format | PCM (raw bytes, no container/headers) |
| Bit depth | 16-bit signed (little-endian) |
| Sample rate | **24,000 Hz** |
| Channels | Mono |
| Encoding | Base64 |
| Chunk size | 600ms first chunk, 1s subsequent |
| Max per packet | 1 MB |

**Wrong sample rate = garbled or silent avatar. No error returned.**

**Best practice: configure your TTS provider to output PCM 24KHz directly.** Most providers (ElevenLabs, OpenAI, Google, Azure) have an output format / sample rate setting. This avoids resampling entirely. For example, ElevenLabs lets you set `output_format: "pcm_24000"` in the API request.

### Python: Send audio

```python
import base64, json

def send_audio(ws, pcm_bytes_24khz, event_id):
    ws.send(json.dumps({
        "type": "agent.speak",
        "event_id": event_id,
        "audio": base64.b64encode(pcm_bytes_24khz).decode()
    }))
    ws.send(json.dumps({
        "type": "agent.speak_end",
        "event_id": event_id
    }))
```

### Python: Resample to 24KHz

```python
import numpy as np

def resample_to_24k(pcm_bytes: bytes, original_rate: int) -> bytes:
    samples = np.frombuffer(pcm_bytes, dtype=np.int16).astype(np.float64)
    new_length = int(len(samples) * 24000 / original_rate)
    indices = np.linspace(0, len(samples) - 1, new_length)
    resampled = np.interp(indices, np.arange(len(samples)), samples)
    return np.clip(resampled, -32768, 32767).astype(np.int16).tobytes()
```

### Node.js: Send audio

```javascript
function sendAudio(ws, pcmBuffer) {
  const eventId = `speak-${Date.now()}`;
  ws.send(JSON.stringify({
    type: 'agent.speak', event_id: eventId,
    audio: pcmBuffer.toString('base64')
  }));
  ws.send(JSON.stringify({
    type: 'agent.speak_end', event_id: eventId
  }));
}
```

### Streaming TTS audio

When streaming TTS output, send chunks under the **same `event_id`** as they arrive. Use a larger first chunk (600ms) to let the avatar buffer, then 1s chunks after that. Send `agent.speak_end` once after the stream ends.

```python
import base64, json
from uuid import uuid4

BYTES_PER_SEC = 48_000  # 24KHz × 16-bit mono = 48,000 bytes/sec
FIRST_CHUNK = int(BYTES_PER_SEC * 0.6)  # 600ms — initial buffer
NEXT_CHUNK = BYTES_PER_SEC              # 1s — subsequent chunks

def stream_audio(ws, tts_stream):
    """Stream TTS audio chunks to LiveAvatar as they arrive."""
    event_id = f"speak-{uuid4()}"
    buffer = b""
    first = True

    for pcm_data in tts_stream:
        buffer += pcm_data
        target = FIRST_CHUNK if first else NEXT_CHUNK

        while len(buffer) >= target:
            chunk, buffer = buffer[:target], buffer[target:]
            ws.send(json.dumps({
                "type": "agent.speak",
                "event_id": event_id,
                "audio": base64.b64encode(chunk).decode()
            }))
            first = False
            target = NEXT_CHUNK

    # Flush remaining audio
    if buffer:
        ws.send(json.dumps({
            "type": "agent.speak",
            "event_id": event_id,
            "audio": base64.b64encode(buffer).decode()
        }))

    ws.send(json.dumps({
        "type": "agent.speak_end",
        "event_id": event_id
    }))
    # Server returns agent.speak_ended when the avatar finishes speaking
```

### Interrupting during a stream

If the user speaks while you're streaming audio, stop sending chunks and send `agent.interrupt`:

```python
import threading

streaming = threading.Event()

def stream_audio(ws, tts_stream):
    event_id = f"speak-{uuid4()}"
    streaming.set()
    buffer = b""
    first = True

    for pcm_data in tts_stream:
        if not streaming.is_set():
            break
        buffer += pcm_data
        target = FIRST_CHUNK if first else NEXT_CHUNK

        while len(buffer) >= target:
            if not streaming.is_set():
                break
            chunk, buffer = buffer[:target], buffer[target:]
            ws.send(json.dumps({
                "type": "agent.speak",
                "event_id": event_id,
                "audio": base64.b64encode(chunk).decode()
            }))
            first = False
            target = NEXT_CHUNK

    if streaming.is_set() and buffer:
        ws.send(json.dumps({
            "type": "agent.speak",
            "event_id": event_id,
            "audio": base64.b64encode(buffer).decode()
        }))

    ws.send(json.dumps({
        "type": "agent.speak_end",
        "event_id": event_id
    }))

def interrupt(ws):
    streaming.clear()  # Stop sending chunks
    ws.send(json.dumps({"type": "agent.interrupt"}))
```

### Keep-alive

Sessions time out after 5 minutes of inactivity. Send a keep-alive every 2-3 minutes:

```python
import time, threading

def keep_alive_loop(ws, interval=120):
    while True:
        time.sleep(interval)
        ws.send(json.dumps({
            "type": "session.keep_alive",
            "event_id": f"keepalive-{uuid4()}"
        }))
```

### Debug: Test tone

If your TTS audio doesn't work, try a known-good sine wave to isolate the problem:

```python
import struct, math
sample_rate = 24000
pcm = b''.join(
    struct.pack('<h', int(32767 * 0.5 * math.sin(2 * math.pi * 440 * i / sample_rate)))
    for i in range(sample_rate)  # 1 second
)
# If this works but TTS doesn't → your TTS format is wrong, resample to 24KHz
```

### Common audio mistakes

| Mistake | Result | Fix |
|---------|--------|-----|
| 16KHz sample rate | Garbled/fast | Resample to 24KHz |
| 44.1KHz sample rate | Garbled/slow | Resample to 24KHz |
| WAV with headers | Clicking at start | Strip headers, send raw PCM |
| MP3/OGG | Silence | Decode to PCM first |
| Not base64 encoded | WebSocket error | Base64 encode before sending |
| Chunks > 1MB | Dropped | Split into ~1s chunks |

## ElevenLabs Agent Plugin (add-on)

Bridges ElevenLabs Conversational AI agents with LiveAvatar video.

### Requirements

- ElevenLabs API key with permissions: `convai_read`, `user_read`, `voices_read`
- ElevenLabs Agent ID
- Agent audio output configured as **PCM 24K**

### Setup

1. **Store key:**
```bash
curl -X POST https://api.liveavatar.com/v1/secrets \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"secret_type": "ELEVENLABS_API_KEY", "secret_value": "<key>", "secret_name": "ElevenLabs Agent Key"}'
```

2. **Session token:**
```json
{
  "mode": "LITE",
  "avatar_id": "<avatar_id>",
  "elevenlabs_agent_config": {
    "secret_id": "<secret_id>",
    "agent_id": "<elevenlabs_agent_id>"
  }
}
```

**Important:** With this plugin, a LiveKit room is auto-created (no `ws_url`). It uses **FULL Mode's event system** (LiveKit data channels), NOT LITE's WebSocket. Don't mix the event systems.

## BYO WebRTC (add-on)

### LiveKit

```json
{
  "mode": "LITE",
  "avatar_id": "<avatar_id>",
  "livekit_config": {
    "url": "wss://your-livekit-server.com",
    "token": "<your_agent_token>"
  }
}
```

### Agora

```json
{
  "mode": "LITE",
  "avatar_id": "<avatar_id>",
  "agora_config": {
    "app_id": "<your_agora_app_id>",
    "channel": "<your_agora_channel>"
  }
}
```

## Sandbox

```json
{"mode": "LITE", "is_sandbox": true, "avatar_id": "dd73ea75-1218-4ef3-92ce-606d5f7fbc0a"}
```

~1 min sessions, no credits.

## Session Teardown

Stop a session when you're done to free resources and stop credit usage:

```bash
curl -X DELETE https://api.liveavatar.com/v1/sessions \
  -H "Authorization: Bearer <session_token>"
```

Close the WebSocket connection after the DELETE call returns. If the WebSocket drops unexpectedly, the session will auto-terminate after the 5-minute inactivity timeout.

## Gotchas

1. **Audio format is king.** PCM 16-bit, 24KHz, base64. Wrong format = garbled with NO error.
2. **Wait for `connected`.** Events before it are silently dropped.
3. **LITE uses WebSocket, not LiveKit data channels.** Don't use `avatar.*` events — those are FULL Mode.
4. **ElevenLabs plugin is a hybrid.** Configured as LITE but uses FULL Mode's event system.
5. **`agent.speak_end` is required.** Without it, avatar won't transition states.
6. **5-minute timeout.** Send `session.keep_alive` every 2-3 minutes.
7. **Same `event_id` for all chunks in one utterance.** Each `agent.speak` chunk within a single response must share the same `event_id`. Using different IDs per chunk will break playback.
8. **Interrupt requires stopping your send loop.** Sending `agent.interrupt` alone isn't enough — you must also stop sending remaining audio chunks, or they'll queue up and play after the interrupt.
9. **Teardown stops credit usage.** Always `DELETE /v1/sessions` when done. Orphaned sessions burn credits until the 5-minute timeout kills them.
