# Gemini CLI Starter Kit

A plug-and-play best practices system for [Gemini CLI](https://github.com/google-gemini/gemini-cli). Structured sessions, automatic logging, production safety, SQL management, and modular rules -- out of the box.

> See the [full guide](GUIDE.md) for detailed usage.

## Quick Start

1. Copy `starter-kit/` contents into your project root.
2. Drop any reference files into `context/`.
3. Run `/start`.

First time? It scans your project, asks smart questions, builds your context, and shows you everything available.
Returning? Loads everything, resumes from your last session. One command.

## What's Inside

```
GEMINI.md              Core rules (~65 lines, imports rules/rules.md)
rules/rules.md         Safety, queries, workflow, logging — one file
CONTEXT.md             Your project context (built by /start)
PREFERENCES.md         Your personal preferences (tone, style, habits)
.geminiignore          Keeps secrets out of context
context/               Reference files -- auto-loaded every session
queries/               SQL query library -- one .sql per query
.gemini/
  settings.json        Checkpointing, compression, safe tool pre-approvals
  commands/            /start, /setup, /session:save, /context:update, /help
  skills/              Skill folders (each with a SKILL.md)
output/                session-log.md, query-log.md, deliverables
```

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume a session |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session -- handoff doc + memory check |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/help` | Show available commands and features |
| `/memory add` | Persist a fact across sessions |
| `/restore` | Undo a file change |

## Example Skills

The `examples/skills/` folder in this repo contains skills you can copy into your project's `.gemini/skills/`:

- **bigquery/** -- BigQuery optimization (partitioning, joins, cost control, bq CLI, schema design, security)
- **tableau-bigquery/** -- Tableau + BigQuery live connections (SQL push-down, cost control, auth, monitoring)

## License

[MIT](LICENSE)
