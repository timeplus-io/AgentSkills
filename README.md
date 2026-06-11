# AgentSkills

A Timeplus-maintained collection of reusable [Claude Code](https://claude.com/claude-code) agent skills, following the [agentskills.io](https://agentskills.io) `SKILL.md` standard. Each top-level directory is one self-contained skill.

## Skills

### Timeplus Streaming SQL & Apps

| Skill | Description |
|-------|-------------|
| [timeplus-sql-guide](timeplus-sql-guide/) | Write and execute Timeplus streaming SQL — streams, streaming queries, materialized views, ingestion, sinks, UDFs, and full DDL/DML. |
| [timeplus-app-builder](timeplus-app-builder/) | Create, package, and install Timeplus apps (`.tpapp`) — manifests, template variables, dashboard JSON, and install debugging. |
| [timeplus-design](timeplus-design/) | Build UIs that match the Timeplus Console look and feel, with design tokens plus ready-to-adapt CSS, Tailwind, and React. |
| [pulsebot-app-builder](pulsebot-app-builder/) | Build real-time, single-file HTML/JS apps in Pulsebot that connect to Timeplus Proton and visualize live data with Vistral. |

### Integrations & Utilities

| Skill | Description |
|-------|-------------|
| [searxng-web-search](searxng-web-search/) | Search the web via a self-hosted SearXNG metasearch instance with the JSON API enabled. |
| [cisco-asa-syslog](cisco-asa-syslog/) | Parse, interpret, and analyze Cisco ASA firewall syslog messages for event analysis and security investigations. |

## Installation

The skills in this repo are published through three channels. Pick whichever matches your tooling.

### Option 1: skills.sh CLI (recommended)

The [skills.sh](https://skills.sh) CLI installs skills straight from this GitHub repo into 20+ agents (Claude Code, Cursor, GitHub Copilot, Windsurf, and more):

```bash
# Interactive — pick the skills and agents you want
npx skills add timeplus-io/AgentSkills

# Install every skill in the repo
npx skills add timeplus-io/AgentSkills --all

# Install a specific skill
npx skills add timeplus-io/AgentSkills --skill timeplus-sql-guide
```

Useful flags:

- `-g` / `--global` — install to `~/<agent>/skills/` instead of the project-local `./<agent>/skills/`
- `-a <agent>` / `--agent <agent>` — target specific agents (e.g. `-a claude-code -a cursor`), or `--agent '*'` for all
- `-y` — non-interactive, for CI/CD

Manage installed skills with `npx skills list`, `npx skills update`, and `npx skills remove`.

### Option 2: agentskill.sh

All skills are published under [agentskill.sh/@timeplus-io](https://agentskill.sh/@timeplus-io). First set up the agentskill.sh integration for your agent:

```bash
# Universal setup for any installed AI agent
npx @agentskill.sh/cli@latest setup
```

Or, in Claude Code specifically:

```
/plugin marketplace add https://agentskill.sh/marketplace.json
/plugin install learn@agentskill-sh
```

Then install a skill by asking your agent to learn it:

```
/learn @timeplus-io/timeplus-sql-guide
```

### Option 3: ClawHub (for OpenClaw)

Skills published to [ClawHub](https://clawhub.ai) (e.g. [timeplus-sql-guide](https://clawhub.ai/gangtao/timeplus-sql-guide)) can be installed with OpenClaw's native skill commands:

```bash
openclaw skills install timeplus-sql-guide
```

Or with the standalone ClawHub CLI, which installs into `./skills` in your current directory:

```bash
clawhub install timeplus-sql-guide
```

### Manual install

Each skill is a self-contained directory, so you can always copy one directly into your agent's skills folder, e.g. for Claude Code:

```bash
git clone https://github.com/timeplus-io/AgentSkills.git
cp -r AgentSkills/timeplus-sql-guide ~/.claude/skills/
```

## Skill layout

Each skill follows the standard structure:

```
<skill-name>/
  SKILL.md       # YAML frontmatter + Markdown guide (required)
  README.md      # Optional, when the skill has its own install/setup story
  references/    # Detailed topic files linked from SKILL.md
  scripts/       # Optional executable helpers
```

The directory name and the `name:` field in `SKILL.md` always match (kebab-case).

## License

Apache-2.0
