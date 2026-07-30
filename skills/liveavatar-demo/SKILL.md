---
name: liveavatar-demo
description: |
  Spin up a working LiveAvatar demo from a curated catalog — clones the reference repo, installs dependencies, provisions the LiveAvatar account, and fills the env file, then hands off the parts only a human can do. Use when: (1) User wants to try, test, see, or demo LiveAvatar, (2) User asks what LiveAvatar demos or examples exist, (3) User wants a sales agent / lead-qualification avatar, (4) User wants a LiveKit agent driving a LiveAvatar avatar, (5) User wants an avatar joining their own LiveKit room, (6) User says "run the LiveAvatar demo", "show me LiveAvatar", "get me started with LiveAvatar", or "clone the LiveAvatar example", (7) User is evaluating LiveAvatar and wants something running before writing their own integration.
license: MIT
metadata:
  author: heygen
  version: "1.1.0"
---

# LiveAvatar Demos

Three curated demos, each a real open-source repo. This skill does every mechanical step: clone, install, provision, configure, launch.

## What this skill cannot do — say this up front

**You cannot verify that the avatar talks.** Every demo ends at a microphone and a human ear. Do not claim success you can't observe, and do not stall waiting for something you can't reach. Name these moments as you hit them:

| Moment | Why it's human-only |
|--------|---------------------|
| Supplying API keys | Cannot be fetched or guessed |
| Demo 1: `npm run setup` prompts | Interactive `readline`. See the reference — running it wrong orphans a context in the user's account |
| Demo 1: fill name + work email, click **Chat with Wayne** | The submit button is disabled until both fields are filled |
| Demos 2/3: grant browser mic permission, then speak | No mic, no conversation |
| Confirming the avatar spoke and lip-synced | Audio + video, in a human's browser |
| Vercel login / deploy (demo 1, Full level only) | Interactive auth |

Everything else is yours. Do it without narrating each command.

## Step 1: Get the LiveAvatar API key — before anything else

All three demos need it, and having it early removes most branching. Ask for it now, in your first message, alongside the catalog:

> Grab an API key from https://app.liveavatar.com → Settings → API keys and paste it here. I'll take it from there.

**Do not proceed past Step 3 without it.** You may clone and install while waiting, so the user is one paste away from a running demo — say that's what you're doing.

Never write a placeholder or fake value into an env file. `setup.mjs` and both Python entrypoints treat whatever is in the file as authoritative, and a bogus key produces a 401 that looks like an account problem.

## Step 2: Pick the Demo

Show this table unless the user already named one.

| # | Demo | What it does | Stack | Extra keys needed |
|---|------|--------------|-------|-------------------|
| 1 | **AI Sales Agent** | Avatar qualifies leads, writes a summary to Notion + Slack on hangup | Next.js 15 / TypeScript, Node 22+ | none for a local run; Anthropic + a public URL for the full sales brain |
| 2 | **LiveKit Agent (LiveAvatar-hosted room)** | Your own LiveKit STT→LLM→TTS pipeline driving the avatar. LiveAvatar owns the room | Python 3.10+ / LiveKit Agents | LiveKit Cloud key + secret |
| 3 | **BYO LiveKit Agent** | Same pipeline, avatar joins a room in **your** LiveKit project. Production-shaped | Python 3.10+ / LiveKit Agents | LiveKit Cloud key + secret + URL, `livekit-cli` |

Routing: fastest thing that talks → **1**. Wants to see a voice pipeline, has LiveKit Cloud → **2**. Already runs LiveKit rooms in production → **3**. "Just show me anything" → **1**.

Demos 2 and 3 are the same repo, two entrypoints. Wanting both means cloning once.

### Costs — state this once, before launching

- LiveAvatar sandbox sessions are free and duration-capped at roughly a minute. Demos 2 and 3 default to sandbox; demo 1 uses the shared public demo avatar.
- **Demos 2 and 3 bill STT/LLM/TTS to the user's LiveKit Cloud project — including demo 2, where LiveAvatar owns the room.** Room ownership and inference billing are separate. People assume otherwise. Demo 3 also bills RTC minutes.
- Demo 1 bills Anthropic tokens per turn, and only once the brain is wired.

## Step 3: Preflight

Run only the checks for the chosen demo. Report anything missing with its install command; don't install runtimes without asking.

**Demo 1:**
```bash
node --version                 # need >= 22 (engines.node enforces it)
git --version
lsof -ti tcp:3003              # MUST be empty — see below
```

Port 3003 is **pinned** in both `dev` and `start`. An occupied port is fatal, not a warning, and there is no auto-increment. If `lsof` returns a PID, either free it or plan to bypass npm with `npx next dev --port <free>` — and remember the handoff URL changes with it.

**Demos 2 and 3:**
```bash
git --version
uv --version                   # install: brew install uv
```

**Do not gate on `python3 --version`.** `uv venv` reads `requires-python` from `pyproject.toml` and selects a compatible interpreter itself — a system Python 3.9 is irrelevant. Checking it produces a false negative that aborts a setup which would have worked. If `uv` is genuinely unavailable, fall back to `python3.13 -m venv .venv && .venv/bin/pip install -e .` and say you did.

Demo 3 also needs `lk`: `brew install livekit-cli` (tap `livekit/livekit` first if it doesn't resolve), then `lk cloud auth` — interactive, hand it to the user.

## Step 4: Remaining Credentials

Ask for these in **one** message, after the demo is chosen — not before, since what's required depends on it.

| Demo | Also needed | Source |
|------|-------------|--------|
| 1, local run | nothing | — |
| 1, full sales brain | `ANTHROPIC_API_KEY` | https://console.anthropic.com |
| 2 | `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | https://cloud.livekit.io → project → Settings → Keys |
| 3 | those two **plus** `LIVEKIT_URL` (`wss://….livekit.cloud`) | same page |

For demo 1, read the Local-vs-Full section of its reference and pick a level **before** asking — asking for an Anthropic key the local run never uses blocks users who have only a LiveAvatar key.

**Avatar ID: never ask.** Both repos ship a working sandbox avatar in `.env.example`. Leave it.

Optional and demo-1-only, skip unless raised: `NOTION_TOKEN` + `NOTION_DATABASE_ID`, `SLACK_WEBHOOK_URL`.

### Where to clone

Default to `../liveavatar-demos/<repo-name>` relative to the current directory, and confirm the path before running `git clone`. Never clone inside the user's project tree — if cwd *is* their project, go up and out. If the target exists and is non-empty, stop and ask.

## Step 5: Set Up and Run

Follow the matching reference. It has the exact commands, env layout, ready signal, and failure modes.

| Demo | Reference |
|------|-----------|
| 1 — AI Sales Agent | [references/sales-agent.md](references/sales-agent.md) |
| 2 — LiveKit hosted room | [references/livekit-agent.md](references/livekit-agent.md) |
| 3 — BYO LiveKit | [references/livekit-agent.md](references/livekit-agent.md) (BYO section) |

Rules for every demo:

**Secrets go in `.env.local`, never `.env`, never a committed file.** Both repos gitignore `.env.local` and ship `.env.example`.

**Write keys with an editor tool, not a shell command.** Shell history and logs persist. The exception is demo 1's `setup.mjs`, which reads `LIVEAVATAR_API_KEY` from the environment — the reference covers that case.

**Copying `.env.example` is not enough.** Both `.env.example` files ship keys with empty values that are *not* equivalent to unset. The references name each one. Getting this wrong is the single most common way these demos die.

**Never activate a venv.** `source .venv/bin/activate` does not persist between your tool calls. Use `uv run <command>` every time.

**Launch long-lived processes in the background and read the log** rather than blocking on a foreground process. Interactive setup scripts are the exception — see the reference.

**Report the ready signal you actually saw, then stop.** Each reference names one. Every ready signal proves the *process* started, not that the avatar works. Do not extrapolate.

## Step 6: Hand Off

Give exactly this, then stop:

1. **What the user must do now**, concretely. Demo 1: open http://localhost:3003, fill in a name and work email, click **Chat with Wayne**. Demos 2/3: a browser tab opened automatically — allow the microphone, then speak.
2. **What you could not verify** — that the avatar appears, speaks, and lip-syncs. Ask them to confirm.
3. **One customization pointer.** Demo 1: prompt parts in `src/lib/ai-sales/brain/prompt-parts/`. Demos 2/3: `instructions=` in `src/agent.py`.
4. **How to stop, and what's billing.** The local dev server costs nothing. The billable thing is the avatar session — it ends when the browser tab closes or the process dies. Say which one applies.

Don't tour the codebase. They'll ask.

## When Things Break

Each reference has a gotchas section with the specific failure. General triage:

1. **Crash before any network call** → an empty-but-present env var. Check `LOG_LEVEL` (demos 2/3) first; it's the most common and its traceback names nothing relevant.
2. **Avatar connects but is silent** → missing context (`opening_text`), or an LLM configuration pointed at an unreachable URL. Silent failure, no error thrown.
3. **Crash on the user's first sentence (demos 2/3)** → model weights never downloaded.
4. **401 at session mint** → bad or missing `LIVEAVATAR_API_KEY`. This error is clear and quotes the request body; trust it.
5. **Session rejected at mint** → `max_session_duration` above the account tier's cap, or a production avatar ID in sandbox mode.
6. **Nothing in the browser** → check the browser console before the server log. Usually mic permission.

If the symptom isn't in the reference's table, debug from the log, not the README.

## Step 7: Collect Feedback

**Trigger once the demo is handed off**, or if the user abandons after 3+ failed attempts at the same step. If you're blocked on credentials and nothing is running, skip it.

Follow the `liveavatar-feedback` skill if it's available; if it isn't, skip silently rather than improvising a feedback channel. One line, and don't push if the user ignores it.

## What to consult

- [references/sales-agent.md](references/sales-agent.md) — AI Sales Agent: interactive setup script, local vs. deployed, Vercel, env reference
- [references/livekit-agent.md](references/livekit-agent.md) — both LiveKit demos: common setup, hosted flow, BYO flow, worker deploy
