# AGENTS.md

This repository contains AI agent skills for integrating with LiveAvatar. The skills follow the [Agent Skills](https://agentskills.io) format.

## Repository Structure

```
liveavatar-agent-skills/
├── README.md                        # User-facing documentation
├── AGENTS.md                        # This file (for AI agents)
├── CLAUDE.md                        # Points to AGENTS.md
├── CONTRIBUTING.md                  # Contribution guidelines
├── package.json                     # For npx skills add
└── skills/
    ├── liveavatar-integrate/         # End-to-end integration builder
    │   ├── SKILL.md                  # Discovery, routing, golden pathways
    │   └── references/
    │       ├── embed-guide.md        # Embed implementation
    │       ├── full-mode-guide.md    # FULL Mode + Custom LLM/TTS/PTT
    │       └── lite-mode-guide.md    # LITE Mode + audio format + plugins
    └── liveavatar-debug/             # Troubleshooting
        └── SKILL.md
```

## Skill Architecture

### Design Principles

1. **Two skills, clear responsibilities.** `liveavatar-integrate` = discovery → recommendation → implementation. `liveavatar-debug` = troubleshooting.
2. **`liveavatar-integrate` is the main skill.** It scans the user's codebase, asks clarifying questions if needed, picks the best pathway, and walks through implementation. All three integration paths (Embed, FULL, LITE) live here as reference guides.
3. **Pit of success.** Always default to the simplest integration. Embed > FULL > LITE.
4. **Gotcha-driven.** Lead with what breaks, not what works. Silent failures are the biggest enemy.
5. **Backend/frontend split is sacred.** Every code block must label where it runs and which auth header to use.
6. **Smaller skills inline everything.** `liveavatar-debug` has no `references/` directory. Only split into references when SKILL.md would exceed ~500 lines.

### Naming Conventions

- **Skill directory names**: lowercase, hyphens (e.g., `liveavatar-integrate`)
- **Skill `name` in frontmatter**: matches directory name exactly
- **`SKILL.md`**: uppercase, the only uppercase filename in a skill directory
- **Reference files**: lowercase with hyphens (e.g., `full-mode-guide.md`)

### SKILL.md Format

```yaml
---
name: skill-name
description: |
  One-line summary. Use when: (1) first trigger, (2) second trigger, etc.
license: MIT
metadata:
  author: heygen
  version: "X.Y.Z"
---
```

Descriptions must be **trigger-rich** — include explicit scenarios and keywords so agents know when to activate.

### Content Principles

- **Code-first** — include copy-pasteable curl commands and code snippets. Developers skim prose and copy code.
- **Progressive disclosure** — SKILL.md has behavioral guidance, references/ has API specifics.
- **Each skill is self-contained** — no cross-skill references. Each must stand alone.
- **Keep SKILL.md under 500 lines** — move details to `references/`.

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` following the format above
2. Add optional `references/` directory if SKILL.md would exceed ~500 lines
3. Add the skill to `README.md` skills table
4. Update `AGENTS.md` structure diagram
5. Do NOT add cross-references to other skills — each skill must stand alone

## Updating Existing Skills

- **API changes**: Update the affected `references/` files first, then SKILL.md if behavioral guidance changed
- **New gotchas**: Add to both the relevant guide's gotchas section and the SKILL.md principles
- **OpenAPI spec**: Fetch the latest from `https://api.liveavatar.com/openapi.json`

## Distribution

| Channel | Command | What it reads |
|---------|---------|---------------|
| Vercel skills.sh | `npx skills add heygen-com/liveavatar-agent-skills` | `skills/*/SKILL.md` auto-discovery |
| Manual | `git clone` + symlink to `~/.claude/skills/` | `skills/*/SKILL.md` |

## LiveAvatar API Conventions

- Base URL: `https://api.liveavatar.com`
- Auth: `X-API-KEY` header for most endpoints, `Bearer <session_token>` for session operations
- Session lifecycle: token → start → join room (always this order)
- FULL Mode events: LiveKit data channels (`agent-control` / `agent-response`)
- LITE Mode events: WebSocket at `ws_url` (`agent.*` / `session.*`)
- Sandbox avatar: `dd73ea75-1218-4ef3-92ce-606d5f7fbc0a` (sessions), `65f9e3c9-d48b-4118-b73a-4ae2e3cbb8f0` (embeds)

## Links

- [LiveAvatar Documentation](https://docs.liveavatar.com)
- [LiveAvatar Dashboard](https://app.liveavatar.com)
- [LiveAvatar Web SDK](https://github.com/heygen-com/liveavatar-web-sdk)
- [Agent Skills Format](https://agentskills.io)
