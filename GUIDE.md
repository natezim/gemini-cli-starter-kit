# Gemini CLI Starter Kit for Data Analysts — Guide

Built for data analysts who work with SQL, Excel, Python, BigQuery, Power BI, and Tableau.
This kit gives Gemini CLI structured sessions, query management, production safety, and
deep domain skills -- out of the box.

## Setup

1. Copy `starter-kit/` contents into your project root.
2. Copy relevant skills from `skills/` into `.gemini/skills/` (Excel, Python, BigQuery, etc.).
3. Drop reference files into `context/` (schemas, docs, specs — auto-loaded every session).
4. Run `/start`. It handles everything — first-time setup and every session after.
5. Run `/info` anytime to see what's available.

Checkpointing is on. Use `/restore` to undo any file change.

## How It Works

**`/start`** is the only command you need. First time, it scans your project (files, .env, queries, skills), asks smart questions, and builds your context. Every time after, it loads everything and asks what you're working on. If you left off mid-project, it picks up from your last handoff.

**Skills activate automatically.** If you ask about BigQuery and the bigquery skill is installed, it loads. Same for Excel, Python, Power BI, Tableau. No manual activation needed.

**Queries are managed.** Write → test → show you → confirm → save. Every execution is logged with row counts and results. Your query library lives in `queries/`.

**`/session:save`** ends a session — writes a handoff doc so next session picks up where you left off.

## What's in the Kit

```
GEMINI.md              Core rules. Imports rules/rules.md.
rules/rules.md         Safety, queries, workflow, logging.
CONTEXT.md             Your project context — built by /start.
PREFERENCES.md         Your preferences — tone, style, habits.
.gemini/settings.json  Checkpointing, compression, safe tool pre-approvals.
.geminiignore          Keeps secrets out of context.

context/               Reference files — auto-loaded every session.
queries/               SQL query library — one .sql per query.
.gemini/commands/      /start, /setup, /session:save, /context:update, /info
.gemini/skills/        Skills that activate based on your work.
output/                session-log.md, query-log.md, deliverables.
```

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume a session |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session — handoff doc + memory check |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/info` | Show available commands and features |
| `/memory add` | Persist a fact across sessions (survives compression) |
| `/restore` | Undo a file change |
| `@./file` | Inject any file into context mid-session |

## Tips

- **context/ is your library.** Drop schemas, docs, specs — auto-loaded, no injection needed.
- **queries/ is your query library.** One .sql per query, iterated in place, execution logged.
- **PREFERENCES.md is your personality file.** Casual? Terse? Funny? Put it there.
- **`/memory add` for critical facts.** Memory survives compression. Chat history doesn't.
- **`/context:update` before closing.** One minute now saves re-explaining next session.

## Sources

- Official Gemini CLI docs (GEMINI.md, /memory, checkpointing, skills, hooks)
- OWASP AI Agent Security Cheat Sheet (risk classification)
- Google Cloud Community PRAR workflow (gated execution)
- HumanLayer CLAUDE.md research (instruction design)
- AGENTS.md standard (cross-tool compatibility)
- LLM limitations research (context management, verification, fabrication prevention)
