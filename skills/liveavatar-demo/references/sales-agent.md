# Demo 1 — AI Sales Agent

**Repo:** https://github.com/heygen-com/liveavatar-sales-agent (MIT, default branch `master`)
**Docs:** https://docs.liveavatar.com/docs/guides/sales-agent
**Stack:** Next.js 15.4, React 19, TypeScript, `@heygen/liveavatar-web-sdk`. Node >= 22.

A single Next.js app: a real-time avatar ("Wayne") that talks to visitors, qualifies leads, and writes them to a CRM. Three pieces:

- **Session** — browser ↔ LiveKit via the web SDK.
- **The brain** — `/api/chat/completions`, an OpenAI-compatible endpoint LiveAvatar calls on every turn. Assembles persona + context, streams Anthropic back as OpenAI-shaped SSE.
- **Session close** — `/api/ai-sales/session-end`: Anthropic summary → Notion upsert → Slack ping. Both integrations env-gated.

## Pick a level first — it changes what you need to ask for

| Level | What the avatar does | Cost to set up |
|-------|---------------------|----------------|
| **Local** | Connects, speaks its opening line, converses using the **account's default LLM** — not this app's sales prompt | LiveAvatar key only. ~2 minutes |
| **Full** | Every turn runs this app's prompt parts through Anthropic | Also needs an Anthropic key and a public URL → deploy first, then provision |

The brain cannot point at `localhost` — LiveAvatar calls it over the public internet. **Default to Local** unless the user asks for real sales behavior or already has Vercel set up.

`ANTHROPIC_API_KEY` is **not required for Local**: `setup.mjs` only warns when it's absent (`ANTHROPIC_API_KEY is not set — the agent cannot think without it`), and the app still connects and converses on the account default. Don't block a Local run on it.

## Local run

```bash
git clone https://github.com/heygen-com/liveavatar-sales-agent
cd liveavatar-sales-agent
npm install                    # ~7s, 491 packages. Audit warnings are expected; ignore them
cp .env.example .env.local
```

Now write `LIVEAVATAR_API_KEY=<key>` into `.env.local` with an editor tool, **before running setup**. `setup.mjs` reads the file first (line 257) and only prompts if the key is absent — pre-writing it removes the first interactive prompt entirely. Leave every other value from `.env.example` as-is; the avatar ID is already the public demo avatar and `AI_SALES_CONTEXT_ID` must stay blank so setup creates one.

### ⚠️ `npm run setup` is interactive twice, and running it wrong orphans a resource

Even with the key pre-written, a **second** prompt fires:

```
setup.mjs:307   POST /v1/contexts                          ← creates a context in the user's account
setup.mjs:329   ask("Public URL of your deployment (blank to skip): ")
setup.mjs:395   writeEnvFile(updates)                      ← context id persisted HERE
```

With stdin closed or piped from `/dev/null`, `rl.question` never settles and **the process exits 0 silently** — after the context was created, before any id was written. Result: an orphaned context in the user's LiveAvatar account, an unchanged `.env.local`, no error message, and every re-run creating another one.

Two safe ways to run it:

```bash
# A — feed the blank answer that skips the URL prompt (Local level)
printf '\n' | npm run setup

# B — supply the URL up front, which bypasses the prompt (Full level)
npm run setup -- --url https://your-app.vercel.app
```

**Verify it wrote through** before moving on: `.env.local` must now contain a non-empty `AI_SALES_CONTEXT_ID`. If it's still blank, a context was created and orphaned — say so, and don't re-run blindly.

If piping fails, this is a legitimate hand-off: ask the user to run `npm run setup` themselves in their terminal and paste the output. It's interactive by design.

Setup is otherwise idempotent — it reuses anything already in `.env.local`, so re-running with the context id present is safe.

### Launch

```bash
npm run dev                    # port 3003, pinned
```

If 3003 is occupied the server dies with `EADDRINUSE` and no fallback. Bypass npm: `npx next dev --port 3011` — and use that port in the handoff.

**Ready signal:** `▲ Next.js 15.4.10` … `✓ Ready in <n>ms` and `- Environments: .env.local`. Note this proves only that Next.js booted — it prints identically with a completely empty `.env.local`. A cheap real check: `curl -s -o /dev/null -w '%{http_code}' localhost:3003` returns 200, and `POST /api/ai-sales/session` returns something other than `{"error":{"message":"Missing required env var: LIVEAVATAR_API_KEY"}}`.

### Hand off

The page is **not** a start button. It renders a lead-capture form — `placeholder="Your name"` and `placeholder="Your work email"` — and `<button type="submit" disabled>Chat with Wayne</button>` stays disabled until both are filled. Tell the user: open the URL, fill in a name and a work email, click **Chat with Wayne**, then confirm whether the avatar appears and speaks.

Repo troubleshooting lives at `docs/PROVISIONING.md`, which has a failure-mode table for symptoms that are hard to diagnose.

## Full run (deploy → provision → set env → redeploy)

Order matters; provisioning needs the URL to already exist. All four steps, including the env paste:

```bash
vercel link                                          # interactive — hand to the user
vercel --prod                                        # first deploy; note the URL
npm run setup -- --url https://your-app.vercel.app   # provisions secret + LLM config, updates .env.local
# → now paste .env.local into Vercel → Project → Settings → Environment Variables (all environments)
#   `.env.local` is gitignored and NEVER deploys with the code. Skipping this is the #1
#   reason a deployed demo behaves differently from local.
vercel --prod                                        # redeploy, now with env vars present
```

`npm run setup -- --url` performs two API calls the user would otherwise hand-roll:

1. `POST /v1/secrets` — `{ secret_type: "OPENAI_API_KEY", secret_value: <AI_SALES_LLM_CONFIG_API_KEY>, secret_name: "<name>-<timestamp>" }` → `secret_id`
2. `POST /v1/llm-configurations` — `{ secret_id, model_name: "<an Anthropic model id>", base_url: "https://<app>/api", display_name: "<name>" }` → `llm_configuration_id`

`secret_name` and `display_name` are **required** — omitting them 422s. Prefer the script over hand-rolling.

## Environment reference

Every var is server-only — the app reads no `NEXT_PUBLIC_*`, so no key reaches the browser. `IS_SANDBOX` does **not** exist in this repo; don't look for it.

### 1. The session — `setup.mjs` fills all of this

| Var | Required | Purpose |
|-----|----------|---------|
| `LIVEAVATAR_API_KEY` | yes | Sent as `X-API-KEY` when minting a session |
| `AI_SALES_AVATAR_ID` | yes | Pre-filled with the public demo avatar `dd73ea75-1218-4ef3-92ce-606d5f7fbc0a` — shared across accounts, works with only the user's own key |
| `AI_SALES_CONTEXT_ID` | yes | Supplies the spoken opening line. **Blank = avatar connects and sits silent, no error** |
| `AI_SALES_AGENT_NAME` / `_ROLE`, `AI_SALES_PRODUCT_NAME`, `AI_SALES_COMPANY_NAME` | no | Agent identity — substituted into the prompt and the UI |
| `AI_SALES_VOICE_ID` | no | Blank = avatar's default voice |
| `AI_SALES_LANGUAGE` | no | Default `en` |
| `AI_SALES_MAX_SESSION_DURATION` | no | Seconds, default 600. Must be <= the account tier's cap or the mint is **rejected** |
| `LIVEAVATAR_API_URL` | no | Default `https://api.liveavatar.com` |

### 2. The brain

| Var | Required | Purpose |
|-----|----------|---------|
| `ANTHROPIC_API_KEY` | Full only | Conversation + session-end summary. Setup warns but does not fail without it |
| `AI_SALES_LLM_CONFIG_API_KEY` | once a configuration exists | Protects this app's `/api/chat/completions`. Any strong random string — `openssl rand -hex 32`. Must match the value registered as the LiveAvatar secret. The route returns 500 while unset rather than serving unauthenticated |
| `LLM_CONFIGURATION_ID` | no | Routes each turn to this app's brain. Blank = account default LLM answers and none of the prompt parts are used |
| `PROMPT_PARTS_DIR` | no | Replace the bundled prompt wholesale |
| `AI_SALES_LEAD_RESOLVER` | no | Lead enrichment. Blank = no-op stub, no lookups |

### 3. After the call

| Var | Required | Purpose |
|-----|----------|---------|
| `AI_SALES_SESSION_SIGNING_SECRET` | yes | HMAC binding session-end to a session this app minted. Fail-closed: while unset, session-end 500s and nothing is written. Setup generates it |
| `NOTION_TOKEN`, `NOTION_DATABASE_ID` | no | CRM upsert on session end |
| `SLACK_WEBHOOK_URL` | no | Slack ping on session end |

## The prompt

Assembled from the `*.md` files in `src/lib/ai-sales/brain/prompt-parts/` (12 of them, `00_prompt_outline.md` … `11_prospect_context.md`), concatenated in filename order.

- Every `*.md` in the directory is a part — adding or removing one needs no code change.
- Files prefixed `_` are ignored (park a part without deleting).
- YAML frontmatter and HTML comments are stripped, so `<!-- … -->` notes never reach the model.
- `{{AGENT_NAME}}`, `{{AGENT_ROLE}}`, `{{PRODUCT_NAME}}`, `{{COMPANY_NAME}}` substitute from the `AI_SALES_*` env vars.
- Several parts ship as scaffolds marked `OPERATOR SCAFFOLD` — where the user adds pricing, competitor handling, customer stories.

Bundled into the serverless functions at build time via `outputFileTracingIncludes` in `next.config.js`, so edits need a redeploy.

## Gotchas

**`npm run setup` with closed stdin exits 0 after creating a context.** See above. The worst failure here because it looks like success.

**Port 3003 is pinned in both `dev` and `start`.** Occupied = fatal, no auto-increment.

**`base_url` is the base, not the route.** `https://your-app.vercel.app/api` — LiveAvatar appends `/chat/completions` itself. Including the route 404s and the agent goes silent after its opening line.

**Use the production domain, not a preview URL.** Preview URLs change per commit, so a configuration pointed at one breaks silently on the next push.

**Env vars don't deploy with code.** `.env.local` is gitignored. Paste into Vercel project settings.

**Blank `AI_SALES_CONTEXT_ID` = silent avatar.** The context carries `opening_text`, spoken before any model is in the loop. For the per-visitor greeting the context's opening text must contain the placeholder `${opening_intro}` — the mint passes the generated line through `dynamic_variables` and LiveAvatar substitutes it at dispatch. Setup creates a context whose opening text is exactly that placeholder.

**A bad key fails cleanly:** `That API key was rejected.` / `GET /v1/avatars?page_size=1 → 401`, exit 1. Trust it.

**Node < 22 fails at install** (`engines: { node: ">=22" }`).

**Repeated setup runs accumulate contexts and secrets** in the user's LiveAvatar account. Check `.env.local` wrote through instead of re-running on a hunch.
