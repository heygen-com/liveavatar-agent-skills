# Demo 4 — GPT-Live × Hyperframes × LiveAvatar

One architecture, one repo, several branches. Each is the same stack — OpenAI **GPT-Live** (full-duplex speech-to-speech) drives the avatar's voice, its tool calls land on screen as animated **Hyperframes** overlays, and **LiveAvatar** renders the face — with a different persona, tool set, and visuals baked in per branch. They install, launch, bill, and break identically. Pick whichever is closest to what the user wants to end up with, because the point of these repos is to be **one-shotted and then iterated on by a coding agent — you.**

**Repo:** https://github.com/heygen-com/liveavatar-gpt-live-demos

| Branch | Ships as | Visuals it demonstrates |
|--------|----------|-------------------------|
| `master` | Japanese tutor for beginners | Term cards, recap panel, avatar shrinking to a corner |
| [`customer-support-demo`](https://github.com/heygen-com/liveavatar-gpt-live-demos/tree/customer-support-demo) | LiveAvatar's own support agent | Pricing sheet, FULL-vs-LITE comparison, contact card |
| [`poker-demo`](https://github.com/heygen-com/liveavatar-gpt-live-demos/tree/poker-demo) | Heads-up poker coach vs. a simulated opponent | Live poker table driven by game state, not by the model's memory |

Closest branch wins: teaching / vocabulary → tutor; product FAQ, pricing, "an avatar that explains our thing" → support agent; game, drill, or any visual driven by state the server owns → poker. Unsure → tutor, simplest to read.

**Stack:** pnpm workspace, TypeScript everywhere. Node >= 20.12 (`engines.node`). Server: Node + ws. Web: Vite + livekit-client + @hyperframes/player. MIT, default branch `master`.

## Read every markdown file in the repo before running anything

This repo is written for agents. After cloning, read **all** of these — they are the setup and iteration source of truth and stay newer than this file:

| File | What it gives you |
|------|-------------------|
| `AGENTS.md` | Commands, env contract, **invariants not to "fix"**, add-a-tool recipe, gotchas. Start here |
| `README.md` | Human overview + **"Troubleshooting a fresh clone"** — the first place to look when it breaks |
| `docs/ARCHITECTURE.md` | Why the pieces are shaped the way they are — read before touching `server/src` |
| `docs/REPURPOSING.md` | Turning it into a different demo: the excavation checklist of every domain-specific hook |
| `docs/ADDING_FRONTEND_COMPONENTS.md` | Full walkthrough: author a Hyperframes composition → wire every hook → callable widget |
| `docs/MAKING_VISUALS_FIRE.md` | The harder half: getting the model to actually *call* the tool in a live conversation |
| `server/prompts/*.md` | The persona itself — `instructions.md` (who), `greeting.md` (how it opens), sometimes `knowledge.md` |

This file holds only what those can't: which branches exist, billing, and the agent-shaped path through setup. Where they disagree, the repo wins.

## OpenAI access — do not gate on a model name

The user needs an `OPENAI_API_KEY` that can reach GPT-Live. `GPT_LIVE_MODEL` ships as a working default in `.env.example` and tracks the current GPT-Live release — **leave them alone and never ask the user to supply or confirm a model id.** The id has changed between releases and will change again; `.env.example` is authoritative, this file is not. The model only matters if the GPT-Live leg fails to authenticate — see gotchas.

## Setup — the agent-shaped path

```bash
git clone -b <branch> https://github.com/heygen-com/liveavatar-gpt-live-demos   # -b master for the tutor
cd liveavatar-gpt-live-demos
pnpm install
cp .env.example .env         # secrets live in .env at the repo ROOT — gitignored; only the server reads it
```

Write `LIVEAVATAR_API_KEY` and `OPENAI_API_KEY` into `.env` with an editor tool. Do not blank or remove the `GPT_LIVE_*` lines the copy brought over — `check-env.mjs` requires `GPT_LIVE_MODEL` to be set. Then verify:

```bash
pnpm run setup               # non-interactive when stdin isn't a TTY: verifies both keys against the live APIs; exit 0 = both good
```

`setup.mjs` is agent-safe, unlike demo 1's: no TTY → clean failure instead of a hang, no account resources created, idempotent writes, keys resolved `.env` → environment → prompt. A rejected key exits 1 naming the dashboard to fix it at. (`pnpm run setup`, never bare `pnpm setup` — the bare form is pnpm's own built-in.)

**Avatar ID: never ask.** A default ships in `.env.example`; unset it and the server picks the first active public avatar and logs which.

### Launch

```bash
pnpm dev                     # background it; the preflight names any missing env var
```

Two processes: server on :8787, Vite on :5173 (proxying `/api` and `/ws` to the server, so the browser sees one origin).

**Ready signal:** `[server] listening on http://localhost:8787` plus Vite's `Local: http://localhost:5173`. Proves the processes booted, nothing more. `.env` is read **only at startup** — restart after any key change.

Ports: Vite moving off 5173 is fine — use the URL it prints. 8787 taken → set `PORT` in `.env` **and** mirror it in `web/vite.config.ts`; the dev proxy target is the only coupling.

## Billing — no sandbox

The server mints sessions with a bare `{ mode: "LITE", avatar_id }` payload: no `is_sandbox`, no `max_session_duration`. That means **production LiveAvatar billing from the moment the session starts**, and the duration cap is whatever the account's default is — nothing in the repo shortens it. The server's own backstops are teardown the instant the browser websocket disconnects, and a 60-second no-mic-audio reaper for a tab that is attached but sending nothing (mic denied, capture died). Neither is a spending limit. The OpenAI side bills GPT-Live audio tokens per turn, plus a backend Responses model on delegated tool turns. Say all of this before launching.

## Hand off

1. Open http://localhost:5173 → **Start** → allow the microphone → talk. Every persona greets and asks if the user is ready before doing anything — the user has to answer out loud. The first overlay should land within the first exchange or two.
2. You cannot verify the avatar speaks, lip-syncs, or that overlays land — ask the user to confirm.
3. **Then offer to iterate — this is what the repo is for.** Three tiers, cheapest first:
   - **Re-skin:** edit `server/prompts/instructions.md` + `greeting.md`, restart the server. Persona only.
   - **New visual:** one tool in `shared/tools.ts` + one Hyperframes composition — follow `AGENTS.md`'s recipe and `docs/ADDING_FRONTEND_COMPONENTS.md`, then `docs/MAKING_VISUALS_FIRE.md` to make the model use it.
   - **New domain:** `docs/REPURPOSING.md`'s excavation checklist names every hook that is domain-specific in code, not prompts. Do it in order.
   Iterate on overlays **without minting a session**: `window.__ui({ widget: "<id>", props: {...} })` in the browser console — widget ids are the keys in `web/src/overlays/index.ts`. Nothing is billed.
4. Stopping: close the tab or kill `pnpm dev`; the billable session ends with them.

## Gotchas beyond the repo's own docs

`README.md`'s troubleshooting section covers the fresh-clone failures (missing env, bad key, silent avatar, mic denied, port conflicts) — use it first. Additions:

- **Avatar appears and blinks but never speaks** → the LiveAvatar leg is up, the GPT-Live leg isn't. Set `GPT_LIVE_DEBUG=1` and restart — it logs every upstream event (audio elided), and a session that authenticates but errors will say why. If the error names the model or an alpha header, the user's key can't reach the model in `.env.example`; the fix is on the OpenAI account side or a fresh `git pull`, not a value you invent.
- **No VAD, no push-to-talk — by design.** The mic streams continuously and the model owns turn-taking. "It never hears me" = mic permission (the status line reads `microphone unavailable`), not a missing VAD to add.
- **No greeting** → an empty `server/prompts/greeting.md` is a feature: it means the user speaks first.
- **One browser per session** — a second websocket for the same id is rejected.
- **Upstream error bodies pass through verbatim** on purpose — a 401 names the real failing key; read the message, it is the real one.
- **Don't "fix" the invariants.** One continuous utterance, no per-turn interrupts, two-step barge-in, silence reconstruction. `AGENTS.md` lists them under "Invariants — do not fix these"; they look like bugs and are not. Same for anything `docs/ARCHITECTURE.md` explains a reason for.
