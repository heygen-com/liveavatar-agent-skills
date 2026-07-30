# Demos 2 & 3 — LiveKit Agent starters (Python)

**Repo:** https://github.com/heygen-com/liveavatar-starter-livekit-agent-python (default branch `master`)
**Docs:** [Custom LiveKit agent](https://docs.liveavatar.com/docs/guides/livekit/custom-livekit-agent) · [BYO LiveKit agent](https://docs.liveavatar.com/docs/guides/livekit/byo-livekit-agent)

One repo, two demos. Both run a LiveKit Agents voice pipeline (STT → LLM → TTS) and tee the TTS audio to LiveAvatar's media server over WebSocket so the avatar lip-syncs it. They differ only in **who owns the LiveKit room**, which determines how the worker ships in production.

| | Demo 2 — LiveAvatar-hosted (Flow 1) | Demo 3 — BYO LiveKit (Flow 2) |
|---|---|---|
| Room owner | LiveAvatar's LK project | The user's LK Cloud project |
| Entrypoint | `src/liveavatar_hosted_demo.py` | `src/byo_livekit_demo.py` |
| `LIVEKIT_URL` | not used — comes back from the API | **required** |
| Processes | 1 | 2 (worker + driver) |
| Prod deploy | self-host a long-lived process (Fly / Render / ECS / k8s). `lk agent deploy` does **not** apply | `lk agent deploy` — LK Cloud runs the worker fleet |
| Walkthrough in repo | `docs/liveavatar-hosted-demo.md` | `docs/byo-livekit-demo.md` |

Shared code: `src/agent.py`, `src/worker.py`, `src/pipeline.py`, `src/avatar_ws.py`, `src/liveavatar_client.py`. Viewer: `viewer/index.html` (vanilla JS, auto-connects via query string).

Sessions are minted in **LITE mode** (`{"mode": "LITE", ...}`) — worth knowing if the user asks how this maps to LiveAvatar's integration modes.

## Common setup — required for both

```bash
git clone https://github.com/heygen-com/liveavatar-starter-livekit-agent-python
cd liveavatar-starter-livekit-agent-python

uv venv                                        # picks a compatible interpreter itself
uv pip install -e .                            # ~30s
uv run -m livekit.agents download-files        # model weights — NOT OPTIONAL
cp .env.example .env.local
```

**Use `uv run` for every subsequent command.** Never `source .venv/bin/activate` — activation does not persist between agent tool calls, and re-sourcing in each call is noise.

**`uv run -m livekit.agents download-files` is the current form.** The repo README still says `python src/worker.py download-files`; that path prints `Invoking the download-files command via your agent script is deprecated as of 1.5.10` and the installed `livekit-agents` is well past that. The old form still works, so don't chase it if a user already ran it.

Expect two alarming-but-harmless warnings during the download: `[transformers] PyTorch was not found. Models won't be available...` and `Warning: You are sending unauthenticated requests to the HF Hub.` The turn detector is ONNX, not PyTorch. **Tell the user these are expected** — "Models won't be available" reads exactly like the weights failing, which is the one failure this step exists to prevent.

Budget for it: about **840 MB** into `~/.cache/huggingface/hub/models--livekit--turn-detector`. Seconds on a warm cache, several minutes cold.

**Skipping `download-files` does not fail at startup.** The agent boots fine and crashes the first time it tries to detect a turn — i.e. the moment the user finishes their first sentence. If that's the symptom, this is the cause.

### Then edit `.env.local`

```bash
LIVEAVATAR_API_KEY=          # required
AVATAR_ID=65f9e3c9-d48b-4118-b73a-4ae2e3cbb8f0   # pre-filled sandbox avatar — leave it
LIVEKIT_API_KEY=             # required
LIVEKIT_API_SECRET=          # required
LIVEKIT_URL=                 # DEMO 3 ONLY (wss://<project>.livekit.cloud)
LOG_LEVEL=INFO               # ⚠️ MUST be set to a real level — see below
```

### ⚠️ `LOG_LEVEL=` empty crashes both demos at import

`.env.example` ships `LOG_LEVEL=` with no value. Both entrypoints do:

```python
root.setLevel(os.environ.get("LOG_LEVEL", "INFO").upper())   # hosted:60, byo:57
```

The key *is* present, so `.get` returns `""` and the `"INFO"` default never fires:

```
ValueError: Unknown level: ''
```

This fires at **line 60, before a single credential is read** — so it looks identical whether the keys are good or not, and the traceback lands three frames into stdlib `logging` without mentioning LiveAvatar, LiveKit, or `.env.local`. **Set `LOG_LEVEL=INFO` (or `DEBUG`) immediately after copying `.env.example`.** Do not leave it blank and do not delete the line without replacing it.

### `IS_SANDBOX` semantics are inverted from intuition

```python
os.environ.get("IS_SANDBOX", "true").lower() == "true"
```

- Line **absent entirely** → sandbox (free, duration-capped). This is the default.
- `IS_SANDBOX=true` → sandbox.
- `IS_SANDBOX=` **blank** → production. Blanking it is not "unset".
- `IS_SANDBOX=false` → production, and the pre-filled sandbox avatar may not exist in the user's account.

`.env.example` ships `IS_SANDBOX=true`, which is correct. Leave it alone for a demo.

Other vars: `LIVEAVATAR_BASE_URL` (blank is safe — the code uses `or`, unlike `LOG_LEVEL`).

**Why a LiveKit Cloud project at all?** The pipeline plugins (`inference.STT/LLM/TTS`) call LiveKit's hosted inference gateway. Both demos bill inference to the user's LK project, **including demo 2** where the room is LiveAvatar's.

Runtime: `pyproject.toml` declares `requires-python = ">=3.10, <3.15"`. On 3.13 the install pulls `audioop-lts` to replace the stdlib `audioop` module removed in that version.

## Demo 2 — LiveAvatar-hosted room

```bash
uv run python src/liveavatar_hosted_demo.py
```

What happens:

1. `POST /v1/sessions/token` then `POST /v1/sessions/start` → `livekit_url`, `agent_token`, `client_token`, `ws_url`.
2. An embedded LiveKit Agents worker starts locally (devmode, unregistered).
3. A single job is dispatched in-process using the pre-minted `agent_token`.
4. A viewer opens in the default browser, auto-connects, and requests the mic.

Only `LIVEAVATAR_API_KEY`, `AVATAR_ID`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` are used. `LIVEKIT_URL` is ignored.

**Ready signal:** the process reaches the session-start log lines and opens a browser tab without exiting. There is **no grep-able "ready" string**, and the viewer is served on an **ephemeral port** (`TCPServer(("127.0.0.1", 0), ...)`) — scrape the actual URL from the log before handing it to the user; don't guess a port.

Then stop and hand off: the avatar appearing, the mic prompt, and whether it speaks are all human observations.

## Demo 3 — BYO LiveKit room

Extra prerequisite:

```bash
brew install livekit-cli      # tap livekit/livekit first if it doesn't resolve
lk cloud auth                 # interactive browser login — hand to the user
```

Two processes:

```bash
# 1 — worker: registers with LK Cloud, accepts dispatches
uv run python src/worker.py dev

# 2 — driver: only after the worker logs that it registered
uv run python src/byo_livekit_demo.py
```

**Start the driver only once the worker is registered**, or the dispatch has nobody to land on and the room comes up with no agent.

What the driver does:

1. Mints a room name + viewer/avatar tokens against the user's LK project.
2. `POST /v1/sessions/token` with `livekit_config={livekit_url, livekit_room, livekit_client_token}` so the avatar joins **their** room as a participant.
3. `POST /v1/sessions/start` → media-server `ws_url`.
4. `AgentDispatchService.create_dispatch(agent_name="my-agent", room=<room>, metadata={"ws_url": ...})`.
5. Opens a viewer pre-filled with `LIVEKIT_URL` + viewer token.

**Ready signal:** worker log shows registration, then a job accepted. Same caveat — process health, not a working avatar.

Deploy the worker to LK Cloud (`Dockerfile` + `livekit.toml` ship in the repo):

```bash
lk agent create --secrets-file .env.local   # first time — writes subdomain + agent id into livekit.toml
lk agent deploy                             # subsequent updates
lk agent status
lk agent logs
```

Demo 3 bills **both** RTC minutes and inference to the user's LK project.

## Customizing

| Change | Where |
|--------|-------|
| System prompt / personality | `instructions=` in `LiveAvatarAgent` (`src/agent.py:28`) |
| STT / LLM / TTS models | `inference.STT/LLM/TTS(...)` in `build_session` (`src/pipeline.py:23`). Model list: https://docs.livekit.io/agents/models/ |
| Tools / function calling | `@function_tool` methods on `LiveAvatarAgent` — example comment at `src/agent.py:41` |
| Avatar appearance | `AVATAR_ID` in `.env.local` |

## Gotchas

**`LOG_LEVEL=` blank → `ValueError: Unknown level: ''` at import.** Both entrypoints. The top failure; fix it before anything else.

**`download-files` skipped → crash mid-sentence, not at startup.**

**Named dispatch has no auto-dispatch fallback (demo 3).** `src/worker.py` sets `AGENT_NAME = "my-agent"` and it must match the `agent_name` passed to `create_dispatch`. A worker relying on default auto-dispatch never receives the job — the room connects and no agent shows up.

**Bad key fails clearly — trust the error.**

```
RuntimeError: create_session_token failed status=401
body='{"code":4001,"data":null,"message":"Invalid API key"}'
request_body={'mode': 'LITE', 'avatar_id': '65f9e3c9-...', 'is_sandbox': True, ...}
```

It doesn't name the env var or file, so point the user at `LIVEAVATAR_API_KEY` in `.env.local`. LiveKit credentials are **not** validated at this point and will fail later, less clearly.

**Always call `stop_session` on exit.** Shutdown callbacks handle it; killing the process ungracefully leaves a session billing. `max_session_duration` is the TTL safety net.

**`LIVEKIT_API_SECRET` and `LIVEAVATAR_API_KEY` are server-side only.** The viewer gets a scoped room token, nothing more.

**Demo 2's inference is still billed to the user.** Room ownership ≠ inference billing.

**No mic, no conversation.** If the avatar appears but never responds, check the browser console before the Python log.

**Cleanup.** The clone plus ~840 MB in `~/.cache/huggingface` outlive the demo. Mention it if the user was only evaluating.
