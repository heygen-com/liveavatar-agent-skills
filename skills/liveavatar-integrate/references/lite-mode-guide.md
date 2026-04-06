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

### Events you receive

| Event | Payload |
|-------|---------|
| `session.state_updated` | `{"state": "connected" \| "connecting" \| "closed" \| "closing"}` |
| `agent.speak_started` | `{"event_id": "...", "task": {"id": "..."}}` |
| `agent.speak_ended` | `{"event_id": "...", "task": {"id": "..."}}` |

## Audio Format

**This is where most LITE integrations break.**

| Parameter | Required value |
|-----------|---------------|
| Format | PCM (raw bytes, no container/headers) |
| Bit depth | 16-bit signed (little-endian) |
| Sample rate | **24,000 Hz** |
| Channels | Mono |
| Encoding | Base64 |
| Chunk size | ~1 second recommended |
| Max per packet | 1 MB |

**Wrong sample rate = garbled or silent avatar. No error returned.**

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
import audioop

def resample_to_24k(pcm_bytes, original_rate, sample_width=2):
    resampled, _ = audioop.ratecv(pcm_bytes, sample_width, 1, original_rate, 24000, None)
    return resampled
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

## Gotchas

1. **Audio format is king.** PCM 16-bit, 24KHz, base64. Wrong format = garbled with NO error.
2. **Wait for `connected`.** Events before it are silently dropped.
3. **LITE uses WebSocket, not LiveKit data channels.** Don't use `avatar.*` events — those are FULL Mode.
4. **ElevenLabs plugin is a hybrid.** Configured as LITE but uses FULL Mode's event system.
5. **`agent.speak_end` is required.** Without it, avatar won't transition states.
6. **5-minute timeout.** Send `session.keep_alive` every 2-3 minutes.
