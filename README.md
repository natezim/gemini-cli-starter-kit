# Gemini CLI Starter Kit

A plug-and-play best practices system for [Gemini CLI](https://github.com/google-gemini/gemini-cli). Structured sessions, automatic logging, production safety, SQL management, and modular rules -- out of the box.

> See the [full guide](GUIDE.md) for detailed usage.

## Quick Start

1. Copy `starter-kit/` contents into your project root.
2. Drop any reference files into `context/`.
3. Run `/init`.

First time? It scans your project, asks smart questions, builds your context.
Returning? Loads everything, shows status, asks what you're doing. One command.

## What's Inside

```
GEMINI.md              Core rules (~70 lines, imports from rules/)
rules/                 safety.md, sql.md, workflow.md
CONTEXT.md             Your project context (built by /init)
settings.json          Checkpointing, compression, safe tool pre-approvals
.geminiignore          Keeps secrets out of context
context/               Reference files -- auto-loaded every session
queries/               SQL query library -- one .sql per query
.gemini/commands/      /init, /setup, /session:save, /context:update
.gemini/skills/        Skill folders (each with a SKILL.md)
output/                session-log.md, query-log.md, deliverables
```

## Commands

| Command | What it does |
|---|---|
| `/init` | Start a session (first-time setup or returning) |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session -- handoff doc + memory check |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/stats` | Check context window usage |
| `/memory add` | Persist a fact across sessions |
| `/skills list` | View discovered skills |
| `/restore` | Undo a file change |

## Example Skills

The `examples/skills/` folder contains skills you can copy into `.gemini/skills/`:

- **bigquery/** -- BigQuery optimization expert (partitioning, clustering, joins, HLL++, cost control, schema design, scripting, 2024-2025 features)
- **tableau-bigquery/** -- Tableau + BigQuery live connections (SQL push-down, cost control, auth, monitoring)

## License

[MIT](LICENSE)
