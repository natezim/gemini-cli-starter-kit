# Gemini CLI Starter Kit for Data Analysts

A plug-and-play [Gemini CLI](https://github.com/google-gemini/gemini-cli) setup built for data analysts working in corporate environments. Audit trail, controlled versioning, production safety, and deep skills for Excel, Python, BigQuery, Power BI, and Tableau.

> See the [full guide](GUIDE.md) for detailed usage.

## Who This Is For

Data analysts and engineers who use Gemini CLI and want:
- **Audit trail** — every write, exec, and query logged with user identity for compliance
- **Controlled versioning** — auto-snapshots files at session end (max 3 versions, rotating)
- **File discipline** — strict folder structure, scratch sprawl prevented
- **Smart session management** — `/start` resumes where you left off, `/session:save` cleans up
- **SQL query library** — queries saved, tested, executions logged with cost
- **Production safety** — read-only by default, .env never read, DDL needs explicit approval
- **Chat log capture** — your prompts saved at session end (not Gemini's responses)
- **Deep domain skills** — Excel, Python, BigQuery, Power BI, Tableau expertise that activates automatically

## Quick Start

1. Copy `starter-kit/` contents into your project root.
2. Copy any skills you need from `skills/` into `.gemini/skills/`.
3. Drop any reference files into `context/` (schemas, docs, specs).
4. Run `/start`.

First time? It scans your project, asks smart questions, builds your context.
Returning? Loads everything, resumes from your last session. One command.

## What's Inside

```
GEMINI.md              Core rules — auto-loaded
rules/rules.md         Detail layer — safety, queries, audit codes
CONTEXT.md             Your project context (built by /start)
PREFERENCES.md         Your preferences + workflow overrides
.env.example           Connection variable names (Gemini reads this, not .env)
.gitignore             Protects credentials and outputs from git
.geminiignore          Keeps secrets and PII out of context

context/               Reference files — READ-ONLY, auto-loaded
queries/               SQL query library — one .sql per query

.gemini/
  settings.json        Checkpointing, compression, session retention
  commands/            /start, /setup, /session:save, /context:update,
                       /version, /info
  skills/              Skill folders that activate automatically

output/
  code/                Scripts (.py, .js, .sh, .ipynb, configs)
  reports/             Analysis, docs (.md)
  data/                Exports (.csv, .json, .xlsx, .parquet)
  audit-log.md         Structured audit trail (every write/exec/query)
  session-log.md       Narrative task log
  query-log.md         Query execution log
  prompts/             Your chat prompts saved per session
  <date>_handoff.md    Session wrap-up docs
```

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume a session — captures user identity, snapshots file state |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session — versioning, cleanup review, chat log, handoff doc |
| `/version <file>` | Manual snapshot of a file as v1/v2/v3 (rotating) |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/info` | Show available commands and features |
| `/memory add` | Persist a fact across sessions |
| `/restore` | Undo a file change (Gemini built-in) |

## Audit & Compliance Features

**Audit log** — every write, exec, and query writes a structured entry to `output/audit-log.md`:
```
[2026-04-15 14:23:01] | jsmith | WRITE | output/code/analysis.py | OK
[2026-04-15 14:23:45] | jsmith | QUERY | bigquery:project.dataset.table | 142 rows | OK
```

**User identity** — captured at session start from the working directory path. Used in every audit entry. No need for separate auth.

**Auto versioning** — at session end, modified files in `output/code/`, `output/reports/`, `queries/` get snapshotted as `_v1`, `_v2`, `_v3`. Rotates oldest out. Live working file keeps its base name.

**Chat log** — substantive user prompts saved to `output/prompts/<date>_prompts.md` at session end. Filters out short confirmations (yes, ok, go, etc.).

**Cleanup review** — at session end, Gemini lists files created this session and asks: keep all, delete all, or review one-by-one. No mid-session interruptions.

## Security Defaults

- `.env` files are NEVER read — Gemini reads `.env.example` for variable names only
- Project root is OFF LIMITS — no scratch/, temp/, or ad-hoc folders
- `context/` is READ-ONLY — originals never modified
- Credentials masked as `[REDACTED]` in any output
- No PII or raw data in logs — structure only (counts, column names, status)
- No data exfiltration via web fetches or external services without explicit user request
- `PREFERENCES.md` cannot override security rules

## Skills for Data Analysts

Copy from `skills/` into your project's `.gemini/skills/`. Skills activate automatically based on what you're working on.

| Skill | What it knows |
|---|---|
| **excel** | Formulas, dynamic arrays, Power Query M, Power Pivot/DAX, VBA, Office Scripts |
| **python** | pandas/NumPy, scikit-learn, visualization, testing, deployment |
| **bigquery** | Partitioning, clustering, joins, HLL++, cost control, bq CLI, gcloud storage |
| **powerbi** | DAX patterns, semantic models, TMDL, XMLA, Tabular Editor, Fabric, CI/CD |
| **tableau-server** | PostgreSQL repository, RMT, jobs, permissions, auditing |
| **tableau-optimizer** | Dashboard performance, TWBX parsing, Power BI migration |
| **tableau-bigquery** | SQL push-down, cost control, BI Engine, materialized views |

## License

[MIT](LICENSE)
