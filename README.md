# LiveAvatar Agent Skills

Reusable skills for AI coding agents integrating with [LiveAvatar](https://liveavatar.com). These skills provide procedural knowledge that helps agents build LiveAvatar integrations correctly on the first attempt.

## Available Skills

| Skill | Description |
|-------|-------------|
| **liveavatar-integrate** | End-to-end integration builder — assesses your existing stack, recommends the optimal path (Embed / FULL / LITE), and guides implementation step by step. |
| **liveavatar-debug** | Symptom-based troubleshooting for silent avatars, garbled audio, auth errors, and more. |

## Installation

### Option 1: Using add-skill CLI (Recommended for multi-agent)

```bash
# Install all skills globally
npx skills add heygen-com/liveavatar-agent-skills -a claude-code -g

# Or install to current project only
npx skills add heygen-com/liveavatar-agent-skills -a claude-code

# Install a specific skill only
npx skills add heygen-com/liveavatar-agent-skills --skill liveavatar-integrate
```

This works with Claude Code, Cursor, Codex, and [other agents](https://github.com/vercel-labs/skills#available-agents).

### Option 2: Manual Installation

```bash
git clone https://github.com/heygen-com/liveavatar-agent-skills.git

# Symlink all skills to personal skills directory (available in all projects)
for skill in liveavatar-agent-skills/skills/*/; do
  ln -s "$(pwd)/$skill" ~/.claude/skills/$(basename "$skill")
done
```

Skills activate automatically when agents detect relevant tasks (e.g., "add a LiveAvatar avatar", "build a conversational avatar", "integrate LiveAvatar").

## Skill Structure

Each skill follows the [Agent Skills](https://agentskills.io) format:

```
skill-name/
├── SKILL.md          # Agent instructions with behavioral guidance
└── references/       # Supporting documentation (API details, code examples)
```

## Quick Reference

| Task | Skill |
|------|-------|
| Build a new LiveAvatar integration | `liveavatar-integrate` |
| Put an avatar on a page (no code) | `liveavatar-integrate` → Embed pathway |
| Build a conversational avatar | `liveavatar-integrate` → FULL Mode pathway |
| Use your own LLM / TTS / full pipeline | `liveavatar-integrate` → FULL or LITE pathway |
| Connect ElevenLabs Agents to an avatar | `liveavatar-integrate` → LITE + ElevenLabs pathway |
| Debug a silent or broken avatar | `liveavatar-debug` |

## Example Prompts

```
"Add a LiveAvatar avatar to my landing page"

"Build a conversational AI avatar for customer support"

"I have my own LLM — integrate it with LiveAvatar"

"Set up LiveAvatar LITE Mode with my Pipecat pipeline"

"My LiveAvatar avatar is silent, help me debug"

"Create a sandbox session to test LiveAvatar"
```

## Requirements

- LiveAvatar API key (get one at [LiveAvatar Dashboard](https://app.liveavatar.com))
- Claude Code CLI (or any skills-compatible agent)

## API Reference

All skills use the LiveAvatar API:
- Base URL: `https://api.liveavatar.com`
- Authentication: `X-API-KEY` header (backend), `Bearer <session_token>` (session ops)
- API Versions: v1 (sessions, avatars, voices, contexts), v2 (embeds)

## Design Philosophy

- **Pit of success**: The correct path is the easiest path. Always default to the simplest integration.
- **Gotcha-driven**: Lead with what breaks. Silent failures (missing context, wrong audio format) are called out before agents hit them.
- **Backend/frontend split**: Every code block is labeled with where it runs and which auth to use.
- **Progressive disclosure**: SKILL.md has guidance and structure. References have API specifics and code examples.

## Contributing

Contributions welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines. When adding or modifying skills:

1. Lead with gotchas — what breaks if you do it wrong?
2. Be explicit about backend vs frontend and which auth header to use
3. Include trigger phrases in the skill description
4. Keep SKILL.md under 500 lines — move details to `references/`

## License

MIT

## Related Resources

- [LiveAvatar Documentation](https://docs.liveavatar.com)
- [LiveAvatar Dashboard](https://app.liveavatar.com)
- [LiveAvatar Web SDK](https://github.com/heygen-com/liveavatar-web-sdk)
- [HeyGen Skills](https://github.com/heygen-com/skills) (parent HeyGen video creation skills)
- [Agent Skills Specification](https://agentskills.io/specification)
