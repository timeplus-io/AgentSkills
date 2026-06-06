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
