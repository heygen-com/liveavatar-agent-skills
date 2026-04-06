# FULL Mode Implementation Guide

LiveAvatar manages ASR + LLM + TTS + WebRTC. You configure, ship, and handle events. 2 credits/minute.

```
User audio --> [LiveAvatar: ASR -> LLM -> TTS -> Video] --> Avatar stream
```

## Prerequisites

1. **API key** — https://app.liveavatar.com. Never expose to frontend.
2. **Context ID** — create one below. **Without a context, the avatar is silent (no error thrown).**
3. **Avatar ID** — dashboard or `GET /v1/avatars`
4. **Voice ID** — optional for video avatars, **required for image avatars**. `GET /v1/voices`

### Create a context (required)

```bash
curl -X POST https://api.liveavatar.com/v1/contexts \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Agent",
    "prompt": "You are a friendly customer support agent for Acme Corp. Help users with billing, account setup, and troubleshooting. Keep responses concise and helpful.",
    "opening_text": "Hi there! How can I help you today?"
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Identifier for this context |
| `prompt` | Yes | System instructions — role, tone, knowledge boundaries, guardrails |
| `opening_text` | Yes | Greeting spoken when the session starts |
| `links` | No | Array of `{"url": "...", "faq": "..."}` — URLs the avatar can reference |

Returns: `{ "data": { "id": "<context_id>", "name": "...", ... } }`

Use the returned `id` as `context_id` in your session token. Contexts are reusable — create once, reference in many sessions.

## Session Lifecycle

### Step 1: Create session token (BACKEND, `X-API-KEY`)

```bash
curl -X POST https://api.liveavatar.com/v1/sessions/token \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "FULL",
    "avatar_id": "<avatar_id>",
    "avatar_persona": {
      "voice_id": "<voice_id>",
      "context_id": "<context_id>",
      "language": "en"
    }
  }'
```

Returns: `{ "data": { "session_id": "...", "session_token": "..." } }`

### Step 2: Start session (BACKEND, `Bearer <session_token>`)

```bash
curl -X POST https://api.liveavatar.com/v1/sessions/start \
  -H "Authorization: Bearer <session_token>"
```

**Use the session token, NOT the API key. This is the most common auth mistake.**

Returns: `{ "data": { "livekit_url": "wss://...", "livekit_client_token": "..." } }`

### Step 3: Join LiveKit room (FRONTEND)

Pass `livekit_url` + `livekit_client_token` to your frontend.

**Quick test (no frontend code):**
```
https://meet.livekit.io/custom?liveKitUrl=<livekit_url>&token=<livekit_client_token>
```

**Production — Web SDK:**
```bash
npm install @heygen/liveavatar-web-sdk
```

**For frontend implementation, clone or read the Web SDK repo:**
https://github.com/heygen-com/liveavatar-web-sdk

The repo contains:
- `packages/js-sdk/` — the SDK source with full TypeScript types (`LiveAvatarSession`, `SessionEvent`, `AgentEventsEnum`, `VoiceChat`, etc.)
- `apps/demo/` — a complete Next.js demo app showing FULL Mode, LITE Mode, and Push-to-Talk

**Key patterns from the demo app:**
1. Backend API route creates session token and returns `session_token` to the client
2. Frontend creates `new LiveAvatarSession(sessionToken)` — the SDK handles `/v1/sessions/start` and LiveKit connection internally
3. Listen for `SessionEvent.SESSION_STREAM_READY`, then call `session.attach(videoElement)` to render
4. Call `session.voiceChat.start()` to enable microphone
5. Use `session.message(text)` (LLM response) or `session.repeat(text)` (verbatim) to make the avatar speak
6. Call `session.stop()` to end

### Auth split

| What | Where | Auth |
|------|-------|------|
| Create session token | Backend | `X-API-KEY` (secret) |
| Start session | Backend | `Bearer <session_token>` |
| Join LiveKit room | Frontend | `livekit_client_token` (safe) |
| Send/receive events | Frontend | LiveKit data channels |
| Stop session | Backend | `Bearer <session_token>` |

## Event System: LiveKit Data Channels

Two topics: `agent-control` (send) and `agent-response` (receive).

### Commands (to `agent-control`)

| Event | Payload | What it does |
|-------|---------|-------------|
| `avatar.interrupt` | none | Stop avatar mid-speech, clear queue |
| `avatar.speak_text` | `{"text": "..."}` | Avatar speaks text verbatim |
| `avatar.speak_response` | `{"text": "..."}` | Avatar generates LLM response, then speaks |
| `avatar.start_listening` | none | Switch to listening state |
| `avatar.stop_listening` | none | Switch to idle state |

### Events (from `agent-response`)

| Event | Payload | What it means |
|-------|---------|--------------|
| `user.transcription` | `{"text": "..."}` | What the user said |
| `avatar.transcription` | `{"text": "..."}` | What the avatar is saying |
| `avatar.speak_started` | none | Avatar started speaking |
| `avatar.speak_ended` | none | Avatar finished speaking |
| `user.speak_started` | none | User started talking |
| `user.speak_ended` | none | User stopped talking |
| `session.stopped` | `{"end_reason": "..."}` | Session ended |

All events: `{ "event_id": "string", "event_type": "string", "session_id": "string", "source_event_id": "string | null" }`

## Session Management

- **Keep-alive:** `POST /v1/sessions/keep-alive` with `Bearer <session_token>` every 2-3 minutes (5-min timeout)
- **Stop:** `POST /v1/sessions/stop` with `Bearer <session_token>`
- **Video quality:** `video_quality` (`very_high`/`high`/`medium`/`low`) + `video_encoding` (`VP8`/`H264`) in session token

## Custom LLM (add-on)

Use your own OpenAI-compatible LLM.

### Setup

1. **Store API key:**
```bash
curl -X POST https://api.liveavatar.com/v1/secrets \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"secret_type": "LLM_API_KEY", "secret_value": "<your_llm_key>", "secret_name": "My LLM Key"}'
```

2. **Create LLM config:**
```bash
curl -X POST https://api.liveavatar.com/v1/llm-configurations \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o", "secret_id": "<secret_id>", "base_url": "https://api.openai.com/v1"}'
```

3. **Add to session token:**
```json
{
  "mode": "FULL",
  "avatar_id": "<avatar_id>",
  "llm_configuration_id": "<config_id>",
  "avatar_persona": { "context_id": "<context_id>", "language": "en" }
}
```

Works with any OpenAI-compatible endpoint (OpenAI, Azure, Anthropic via proxy, local models). `base_url` defaults to OpenAI if omitted.

## Custom TTS (add-on)

Use your own ElevenLabs voice.

### Setup

1. **Store ElevenLabs key:**
```bash
curl -X POST https://api.liveavatar.com/v1/secrets \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"secret_type": "ELEVENLABS_API_KEY", "secret_value": "<elevenlabs_key>", "secret_name": "ElevenLabs Key"}'
```

2. **Import voice:**
```bash
curl -X POST https://api.liveavatar.com/v1/voices/third_party \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"secret_id": "<secret_id>", "voice_id": "<elevenlabs_voice_id>"}'
```

3. **Use the returned LiveAvatar `voice_id`** (not the ElevenLabs ID) in `avatar_persona.voice_id`.

## Push-to-Talk (add-on)

Set `interactivity_type: "PUSH_TO_TALK"` in the session token. Then send:
- `user.start_push_to_talk` — begin capturing audio
- `user.stop_push_to_talk` — stop capturing audio

Audio outside PTT windows is ignored.

## Sandbox

```json
{
  "mode": "FULL",
  "is_sandbox": true,
  "avatar_id": "dd73ea75-1218-4ef3-92ce-606d5f7fbc0a",
  "avatar_persona": { "voice_id": "<voice_id>", "language": "en" }
}
```

~1 min sessions, no credits.

## Gotchas

1. **No `context_id` = silent avatar.** Streams video, ignores speech. No error. #1 failure cause.
2. **Wrong auth on `/sessions/start`.** Use `Bearer <session_token>`, not `X-API-KEY`.
3. **API key in frontend.** Must never leave your backend.
4. **Image avatars have no voice.** Must specify `voice_id` or session fails.
5. **5-minute timeout.** Send keep-alive every 2-3 minutes.
6. **Deleting secrets breaks dependents.** No cascade warning. Rotate: new → update → delete old.
