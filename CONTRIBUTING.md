# Contributing to LiveAvatar Agent Skills

Thanks for your interest in contributing! This project provides reusable skills that help AI coding agents integrate LiveAvatar into applications.

## How to Contribute

### Improving Existing Skills

The most valuable contributions improve skill content — making guidance clearer, fixing incorrect advice, or adding missing gotchas that developers commonly hit.

1. Fork and clone the repository
2. Create a branch from `master`
3. Make your changes
4. Open a pull request

### Adding a New Skill

New skills should follow the existing structure:

```
skills/
└── your-skill-name/
    ├── SKILL.md          # The skill content
    └── references/       # Supporting documentation (optional)
```

Every skill must include YAML frontmatter:

```yaml
---
name: skill-name-kebab-case
description: >-
  Trigger phrases and brief description that help agents
  recognize when to activate this skill.
license: MIT
metadata:
  author: heygen
  version: "0.1.0"
---
```

### Filing Issues

- **Bug reports**: Skill content that causes agents to produce incorrect integrations
- **Skill requests**: Ideas for new skills (e.g., framework-specific integration guides)
- **Gotcha reports**: Integration mistakes you've seen that skills should warn about

## Skill Content Principles

### Pit of success

Skills should make the correct path the easiest path. Always default to the simplest integration that meets the user's needs. Never suggest LITE Mode when FULL Mode would suffice.

### Gotcha-driven

Lead with what breaks, not what works. The happy path is in the official docs. Skills exist to prevent the sad path — silent failures, wrong auth headers, garbled audio.

### Backend/frontend split

Every code block should be labeled with where it runs (backend vs frontend) and which auth it uses. The API key must never appear in frontend code.

### Stay under 500 lines

SKILL.md files are loaded into agent context windows. Keep them concise. Move detailed reference material to `references/` files.

### Code-first

Include copy-pasteable curl commands and code snippets. Developers skim prose and copy code.

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE).
