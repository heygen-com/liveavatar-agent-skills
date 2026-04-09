---
name: liveavatar-integrate
description: |
  Build a LiveAvatar integration end-to-end — assesses the user's existing stack, recommends the optimal path, and guides implementation. Use when: (1) Building a new LiveAvatar integration, (2) Adding a real-time avatar to an app or site, (3) Connecting LiveAvatar to an existing AI pipeline, (4) User mentions LiveAvatar, real-time avatar, interactive avatar, conversational avatar, or lip-sync avatar, (5) Deciding between Embed, FULL Mode, and LITE Mode, (6) Migrating from HeyGen Interactive Avatar to LiveAvatar.
license: MIT
metadata:
  author: heygen
  version: "1.0.0"
---

# LiveAvatar Integration

LiveAvatar gives your product a human face — real-time, lip-synced video avatars that speak, react, and maintain eye contact. This skill assesses what you have, recommends the best integration path, and walks you through building it.

## Step 1: Discover What the User Has

Before recommending a path, gather context. Check the codebase and conversation for signals. **Do not ask questions the codebase already answers.**

### Signals to look for in the codebase

Scan for these automatically — do not ask the user if you can detect them:

| Signal | Where to look | What it means |
|--------|--------------|---------------|
| OpenAI / Anthropic / LLM SDK imports | `package.json`, `requirements.txt`, imports | User has their own LLM |
| ElevenLabs / PlayHT / Deepgram TTS SDK | dependencies, imports | User has their own TTS |
| Deepgram / Whisper / AssemblyAI STT SDK | dependencies, imports | User has their own STT |
| LiveKit SDK (`livekit-server-sdk`, `@livekit/`) | dependencies | User has LiveKit infra |
| Agora SDK | dependencies | User has Agora infra |
| Pipecat imports | dependencies, imports | User has a Pipecat pipeline |
| ElevenLabs Agent / Conversational AI | dependencies, config | User has an ElevenLabs agent |
| `HEYGEN_API_KEY` / `LIVEAVATAR_API_KEY` | `.env`, config files | User already has an API key |
| Existing LiveAvatar code | imports, API calls to `api.liveavatar.com` | Existing integration (debug, not new setup) |
| No backend / static site | file structure (pure HTML/CSS/JS, no server) | Embed is the only option |

### Questions to ask (only what's still unknown)

If the codebase scan leaves gaps, ask the user. Frame as a concise checklist — **do not ask these one at a time**:

```
To recommend the best LiveAvatar integration for your setup, I need to know:

1. **What's the goal?** (e.g., customer support avatar, sales demo, onboarding guide, talking head on landing page)
2. **Do you have your own AI pipeline?** (STT, LLM, TTS — or any combination)
3. **Do you need programmatic control** over the conversation (events, interrupts, custom logic), or just an avatar on a page?
```

Skip any question the codebase or conversation already answered.

## Step 2: Route to the Golden Pathway

Based on what you've gathered, match to ONE pathway. **Always pick the simplest path that works.** Do not offer multiple options — make the call.

### Decision tree

```
Has NO backend OR just wants an avatar on a page?
  → EMBED

Has NO existing AI stack (no STT, no LLM, no TTS)?
  → FULL MODE (standard)

Has their OWN LLM but no STT/TTS?
  → FULL MODE + Custom LLM

Has their OWN LLM + their own ElevenLabs TTS?
  → FULL MODE + Custom LLM + Custom TTS

Needs explicit mic control (walkie-talkie style)?
  → FULL MODE + Push-to-Talk

Has a COMPLETE pipeline (STT + LLM + TTS)?
  → LITE MODE

Has an ElevenLabs Conversational AI agent?
  → LITE MODE + ElevenLabs Plugin

Has their own LiveKit or Agora infrastructure?
  → LITE MODE + BYO WebRTC
```

### Golden pathways (pick one, then implement)

| Pathway | When | Implementation guide |
|---------|------|---------------------|
| **Embed** | No backend, or no custom logic needed | [references/embed-guide.md](references/embed-guide.md) |
| **FULL standard** | No existing AI stack | [references/full-mode-guide.md](references/full-mode-guide.md) |
| **FULL + Custom LLM** | Has own LLM, wants LiveAvatar's ASR + TTS | [references/full-mode-guide.md](references/full-mode-guide.md) (Custom LLM section) |
| **FULL + Custom TTS** | Has own ElevenLabs voice | [references/full-mode-guide.md](references/full-mode-guide.md) (Custom TTS section) |
| **FULL + Push-to-Talk** | Needs explicit mic control | [references/full-mode-guide.md](references/full-mode-guide.md) (Push-to-Talk section) |
| **LITE standard** | Has complete STT + LLM + TTS pipeline | [references/lite-mode-guide.md](references/lite-mode-guide.md) |
| **LITE + ElevenLabs Plugin** | Has ElevenLabs Conversational AI agent | [references/lite-mode-guide.md](references/lite-mode-guide.md) (ElevenLabs Plugin section) |
| **LITE + BYO WebRTC** | Has own LiveKit / Agora | [references/lite-mode-guide.md](references/lite-mode-guide.md) (BYO WebRTC section) |

## Step 3: Present the Recommendation

Once you've picked a pathway, tell the user what you recommend and why, in 2-3 sentences. Example:

> Based on your setup, I recommend **FULL Mode with Custom LLM**. You already have an OpenAI integration for your LLM, so we'll plug that in and let LiveAvatar handle ASR, TTS, and video. This gets you a conversational avatar without rebuilding your audio pipeline.

Then proceed directly to implementation using the corresponding guide in `references/`.

## Step 4: Implement

Read the appropriate reference guide and implement. Every guide follows the same structure:

1. **Prerequisites** — what to create/gather before writing code
2. **Session lifecycle** — step-by-step with curl commands and code
3. **Events** — what to send and receive
4. **Add-ons** — mode-specific optional features
5. **Sandbox testing** — free testing before going live
6. **Gotchas** — what breaks and how to avoid it

### Principles that apply to ALL paths

**Backend / frontend split is non-negotiable.** `X-API-KEY` is a secret — backend only. Frontend only gets `livekit_client_token` (safe for browsers). If you see the API key in client code, stop and restructure.

**Context makes the avatar conversational.** In FULL Mode, no `context_id` = silent avatar. No error thrown. Always create a context first, even a minimal `"You are a helpful assistant."`.

**FULL and LITE are completely different protocols.** FULL = LiveKit data channels (`avatar.*` / `user.*`). LITE = WebSocket (`agent.*` / `session.*`). Never mix them.

**Start with sandbox.** `is_sandbox: true`, avatar ID `dd73ea75-1218-4ef3-92ce-606d5f7fbc0a`. Free, ~1 min sessions. Swap to production avatar when ready.

### LITE Mode: Fitting into an existing pipeline

LITE users almost always have a working conversational system already. **Do not ask them to rebuild their pipeline.** Instead, map their existing components onto the LITE turn cycle:

1. **Identify their current flow.** Read their code to understand how conversation turns work today — where does user audio come in, how does it reach the LLM, how does TTS output get delivered? Look for their event loop, message handler, or turn manager.
2. **Find the integration points.** You need to hook into three moments in their existing flow:
   - **User starts/stops speaking** → add `agent.start_listening` / `agent.stop_listening`
   - **TTS produces audio** → route PCM output to `agent.speak` chunks over WebSocket instead of (or in addition to) their current audio output
   - **Response finishes** → send `agent.speak_end` and wait for `agent.speak_ended`
3. **Adapt, don't replace.** If they have a working turn manager, add LiveAvatar calls into it. If they stream TTS to a browser via WebSocket already, tap into that same stream. The goal is the minimum change to get avatar video synced to their existing audio flow.
4. **Verify audio format last.** Once the wiring is in place, confirm their TTS outputs PCM 16-bit 24KHz. If not, either configure the TTS provider's output format or add resampling at the integration point.

## What to consult

- [references/embed-guide.md](references/embed-guide.md) — Embed implementation
- [references/full-mode-guide.md](references/full-mode-guide.md) — FULL Mode implementation + events + add-ons
- [references/lite-mode-guide.md](references/lite-mode-guide.md) — LITE Mode implementation + events + audio format + add-ons
