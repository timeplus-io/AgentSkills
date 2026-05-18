# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Timeplus-maintained collection of reusable Claude Code agent skills, published to ClawHub. Each top-level directory is one self-contained skill (currently: `cisco-asa-syslog`, `searxng-web-search`, `timeplus-app-builder`, `timeplus-sql-guide`). There is no root build, test, or lint step — skills are documentation plus optional reference files and scripts.

## Skill layout

Every skill follows the agentskills.io SKILL.md standard:

```
<skill-name>/
  SKILL.md           # YAML frontmatter + Markdown guide (required)
  README.md          # Optional, only when the skill has its own install/setup story
  references/        # Detailed topic files linked from SKILL.md
  scripts/           # Optional executable helpers (e.g. Python CLI)
```

- Directory name and the `name:` field in SKILL.md frontmatter must match (kebab-case).
- SKILL.md indexes the `references/` files in a quick-reference table near the top — keep that table in sync when adding or renaming references.
- Long content belongs in `references/`, not inline in SKILL.md.

## SKILL.md frontmatter conventions

Required: `name`, `description`. Use these additional fields consistently with existing skills:

- `license: Apache-2.0`
- `compatibility:` — runtime requirements (Python version, required binaries like `curl`, env vars, external services)
- `metadata.author: timeplus-io`
- `metadata.version:` — semantic version, bump on user-visible changes
- `metadata.tags:` — feature keywords for discovery
- `metadata.openclaw:` — only for skills published to OpenClaw/PulseBot; declares `env`, `binaries`, and `primaryEnv` so the host can provision the skill. See `searxng-web-search/SKILL.md` and `timeplus-sql-guide/SKILL.md` for the format.

## Adding a new skill

1. Create `<skill-name>/SKILL.md` with the frontmatter pattern above; match the directory name.
2. Put detailed docs in `<skill-name>/references/*.md` and index them in SKILL.md.
3. If the skill needs runtime tooling, list it under `compatibility:` and (for OpenClaw publishing) `metadata.openclaw:`.
4. Environment variables are the skill's responsibility — document them in SKILL.md and in `compatibility:`. There is no shared `.env.example`.
5. Don't add a top-level README, package manifest, or CI config — each skill stays standalone.

## Repo etiquette

- Remote: `git@github.com:timeplus-io/AgentSkills.git`. Main branch: `main`. Recent history shows feature branches merged via PR (e.g., `feature/...`, `bugfix/...`).
- One skill per PR when practical; bump that skill's `metadata.version` in the same change.
