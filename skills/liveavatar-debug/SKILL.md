---
name: liveavatar-debug
description: "Troubleshoot and debug LiveAvatar integration issues. Use when the user's LiveAvatar integration isn't working, the avatar is silent, audio is garbled, sessions fail to start, events aren't received, or they're getting API errors."
license: MIT
metadata:
  author: heygen
  version: "1.0.0"
---

# LiveAvatar Debugging Guide

Symptom-based troubleshooting for LiveAvatar integrations. Find your symptom below and follow the fix.

## Avatar is silent / not responding to speech

**Most likely: Missing `context_id` in FULL Mode.**

Without a `context_id`, the avatar enters restricted mode — streams video but ignores user input. No error thrown.

**Fix:** Create a context via `POST /v1/contexts` with at least a `prompt` field. Include the returned `context_id` in your session token's `avatar_persona`.

**Other causes:**
- Browser mic permissions blocked (check `allow="microphone"` on iframe)
- Push-to-Talk enabled but PTT events not being sent
- OS-level mic mute

## Audio is garbled or distorted (LITE Mode)

**Most likely: Wrong audio format.**

Required: PCM 16-bit signed, 24KHz, base64. No error returned for wrong format.

**Checklist:**
- [ ] Sample rate is exactly 24,000 Hz
- [ ] Raw PCM (no WAV/MP3/OGG headers)
- [ ] 16-bit signed, not 8-bit or 32-bit float
- [ ] Base64 encoded
- [ ] Chunks under 1MB

**Quick test:** Send a known-good 440Hz test tone. If it works but your TTS doesn't, resample your TTS output.

## 401 error / session fails to start

**Most likely: Wrong auth header.**

| Endpoint | Correct auth |
|----------|-------------|
| `POST /v1/sessions/token` | `X-API-KEY: <api_key>` |
| `POST /v1/sessions/start` | `Authorization: Bearer <session_token>` |

Common mistakes: using API key on `/start`, using Bearer token on `/token`, putting API key in Bearer format.

## Events not received (FULL Mode)

- Subscribed to correct topics? Send to `agent-control`, receive from `agent-response`
- LiveKit room connected? Check connection state
- Parsing JSON correctly? Events have `event_type`, `event_id`, `session_id`

## WebSocket events silently dropped (LITE Mode)

**Most likely: Sending before `connected`.**

Wait for `{"type": "session.state_updated", "state": "connected"}` before sending any commands.

Also check:
- Using correct event names? LITE uses `agent.*`, not `avatar.*` (that's FULL)
- WebSocket still open? 5-min inactivity timeout

## Session times out

5 minutes of inactivity kills the session.

- **FULL:** `POST /v1/sessions/keep-alive` with `Bearer <session_token>`
- **LITE:** `{"type": "session.keep_alive", "event_id": "..."}` via WebSocket

Send every 2-3 minutes.

## Sandbox won't start

- [ ] `is_sandbox: true` set in session token
- [ ] Avatar ID is `dd73ea75-1218-4ef3-92ce-606d5f7fbc0a` (sessions) or `65f9e3c9-d48b-4118-b73a-4ae2e3cbb8f0` (embeds)
- [ ] Not using a different avatar with sandbox mode

~1 minute auto-termination is expected behavior.

## Image avatar fails / no audio

Image avatars have no auto-generated voice. Specify `voice_id` in `avatar_persona`. Browse voices at `GET /v1/voices`.

## Custom LLM / TTS not working

**LLM checklist:**
- [ ] Secret: `secret_type: "LLM_API_KEY"`
- [ ] Config: correct `model`, `secret_id`, `base_url`
- [ ] `llm_configuration_id` in session token
- [ ] Endpoint supports OpenAI `/chat/completions` protocol

**TTS checklist:**
- [ ] Secret: `secret_type: "ELEVENLABS_API_KEY"`
- [ ] Voice imported via `POST /v1/voices/third_party`
- [ ] Using the **returned LiveAvatar voice_id** (not the ElevenLabs ID)

## CORS / network errors

- API calls should come from backend, not browser
- Allow outbound `wss://` for LiveKit and WebSocket
- Allow `*.livekit.cloud`, `api.liveavatar.com`, `embed.liveavatar.com`

## General approach

1. Start with sandbox mode to eliminate billing issues
2. Check session state: `GET /v1/sessions/{id}`
3. Check transcript: `GET /v1/sessions/{id}/transcript`
4. Check credits: `GET /v1/users/credits`
5. Verify you're not mixing FULL and LITE event systems
