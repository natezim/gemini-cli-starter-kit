# Gemini CLI Starter Kit for Data Analysts

A plug-and-play [Gemini CLI](https://github.com/google-gemini/gemini-cli) setup built for data analysts. SQL management, query testing, production safety, session continuity, and deep skills for the tools you actually use -- Excel, Python, BigQuery, Power BI, and Tableau.

> See the [full guide](GUIDE.md) for detailed usage.

## Who This Is For

Data analysts who use Gemini CLI and want:
- **Smart session management** -- `/start` sets up your project, resumes where you left off
- **SQL query library** -- queries saved, tested, versioned in place, execution logged with cost
- **Production safety** -- read-only by default, no `SELECT *`, DDL requires explicit approval
- **Automatic logging** -- every task and query logged without you asking
- **Deep domain skills** -- Excel, Python, BigQuery, Power BI, Tableau expertise that activates automatically

## Quick Start

1. Copy `starter-kit/` contents into your project root.
2. Copy any skills you need from `skills/` into `.gemini/skills/`.
3. Drop any reference files into `context/` (schemas, docs, specs).
4. Run `/start`.

First time? It scans your project, asks smart questions, builds your context.
Returning? Loads everything, resumes from your last session. One command.

## What's Inside

```
GEMINI.md              Core rules (~74 lines, imports rules/rules.md)
rules/rules.md         Safety, queries, workflow, logging
CONTEXT.md             Your project context (built by /start)
PREFERENCES.md         Your preferences (tone, style, habits)
.geminiignore          Keeps secrets out of context
context/               Reference files -- auto-loaded every session
queries/               SQL query library -- one .sql per query
.gemini/
  settings.json        Checkpointing, compression, safe tool pre-approvals
  commands/            /start, /setup, /session:save, /context:update, /info
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
| `/info` | Show available commands and features |
| `/memory add` | Persist a fact across sessions |
| `/restore` | Undo a file change |

## Skills for Data Analysts

Copy from `skills/` into your project's `.gemini/skills/`. Skills activate automatically based on what you're working on.

| Skill | What it knows |
|---|---|
| **excel** | Formulas, dynamic arrays, Power Query M, Power Pivot/DAX, VBA, Office Scripts |
| **python** | pandas/NumPy, scikit-learn, visualization (matplotlib/Plotly/Altair), testing, deployment |
| **bigquery** | Partitioning, clustering, joins, HLL++, cost control, bq CLI, gcloud storage |
| **powerbi** | DAX patterns, semantic models, TMDL, XMLA, Tabular Editor, Fabric, CI/CD |
| **tableau-server** | PostgreSQL repository, RMT, jobs, permissions, auditing, admin dashboards |
| **tableau-optimizer** | Dashboard performance, TWBX parsing, Power BI migration, complexity scoring |
| **tableau-bigquery** | SQL push-down, cost control, BI Engine, materialized views, monitoring |

## License

[MIT](LICENSE)
