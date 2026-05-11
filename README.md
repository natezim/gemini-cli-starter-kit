# Gemini CLI Starter Kit for Data Analysts

A plug-and-play [Gemini CLI](https://github.com/google-gemini/gemini-cli) setup for data analysts in corporate environments. **13 specialized subagents**, deep skills for the BI stack, audit trail, smart session-resume, production safety.

> See the [full guide](GUIDE.md) for usage details, [agent cheatsheet](AGENTS.md) for which subagent to invoke when.

## Install (v2 — two folders, two destinations)

The kit ships as two folders that go to different places:

| Folder | Destination | When |
|---|---|---|
| `global/` | `~/.gemini/` | **Once per machine** — install all rules, agents, skills here |
| `project-template/` | `<your-project>/` | **Per new project** — copy the template into each project root |

### One-time global install

```powershell
# Windows PowerShell
Copy-Item -Recurse global\* "$env:USERPROFILE\.gemini\"
```

```bash
# macOS/Linux
cp -r global/* ~/.gemini/
```

That installs `GEMINI.md` (rules), `PREFERENCES.md`, `settings.json`, all 13 custom agents, all 7 skills, and the custom slash commands. Used by every project on this machine.

### Per-project setup

In each new project root:

```powershell
Copy-Item -Recurse <kit-repo>\project-template\* .
```

Then run `/start` in Gemini CLI. It will identify you, scan the project, ask only what it can't infer, and build out `CONTEXT.md`.

### Updating later

Pull the latest from this repo, copy `global/*` to `~/.gemini/` again. All projects pick up the new rules on next `/start`. Existing projects' `CONTEXT.md` and `MEMORY.md` are untouched.

## Who This Is For

Data analysts and engineers who use Gemini CLI and want:

- **13 specialized subagents** running in isolated context windows so heavy work doesn't bloat the main session
- **Deep domain skills** — Excel, Python, BigQuery, Power BI, Tableau (server / optimizer / BigQuery integration)
- **Audit trail** — every write, exec, and query logged with user identity
- **Smart session resume** — come back in 2 months, `/start` highlights stale threads, agent picks up without re-explaining
- **Controlled versioning** — auto-snapshots files at session end (max 3, rotating)
- **File discipline** — strict folder structure
- **Plan Mode + gated execution** — Pro researches, Flash implements, you approve risky work
- **Production safety** — read-only by default, `.env` never read, DDL needs explicit approval
- **3-tier memory with recurrence gate** — agent persists learnings, but only after they re-appear across sessions (prevents fabricated facts polluting permanent memory)

## What's in the Kit

### `global/` (installs to `~/.gemini/`)

```
GEMINI.md              Global rules — file discipline, audit, security, etc.
PREFERENCES.md         User preferences template
settings.json          Checkpointing, Plan Mode, compression, file auto-load list
commands/              /start, /session:save, /version, /info
agents/                13 specialized subagents
skills/                7 domain skills (excel, python, bigquery, powerbi, 3× tableau)
```

### `project-template/` (copy into each new project)

```
CONTEXT.md             Project description (you write or /start helps you write)
MEMORY.md              Active threads + Learned (agent-managed, you prune)
.gitignore             Sensible defaults
.geminiignore          What Gemini shouldn't see (secrets only, not its own work)
.env.example           Connection variable names template
context/               Drop schemas, docs, specs here (READ-ONLY, auto-loaded)
output/                Workspace
  queries/             SQL library — one .sql per query
  code/                Scripts (.py, .js, .sh, .ipynb, configs)
  reports/             Analyses, docs (.md)
  data/                Exports (.csv, .json, .xlsx, .parquet)
  temp/                Throwaway test/debug — auto-wiped at session end
```

## Subagents (13 total)

Invoke explicitly with `@name`. Read-only agents are parallel-safe.
See [AGENTS.md](AGENTS.md) for the decision tree.

| Agent | Role | Mode |
|---|---|---|
| `@solution-designer` | Brainstorm, decompose, weigh tradeoffs | read-only |
| `@query-validator` | SQL syntax + dry-run cost check | read-only |
| `@schema-explorer` | Schema Q&A from context/ without DDL dump | read-only |
| `@data-profiler` | Shape, nulls, cardinality | read-only |
| `@stats-advisor` | Statistical method + assumptions + library calls | read-only |
| `@sas-migrator` | SAS DATA/PROC/macros → Python | read-only |
| `@excel-reviewer` | Formulas, dynamic arrays, Power Pivot | read-only |
| `@powerquery-reviewer` | M language review, query folding | read-only |
| `@dax-reviewer` | DAX measure correctness + perf | read-only |
| `@tableau-auditor` | Tableau Server inventory, usage, permissions | read-only |
| `@viz-optimizer` | TWBX/PBIP performance issues, ranked | read-only |
| `@migration-mapper` | Tableau ↔ Power BI migration plan | read-only |
| `@report-drafter` | Markdown reports to `output/reports/` | write-scoped |

Built-ins: `@generalist`, `@codebase_investigator`, `@cli_help`.

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume — captures user identity, reads handoff, highlights stale threads |
| `/session:save` | End session — versioning, MEMORY.md update, handoff diff |
| `/version <file>` | Manual snapshot of a file as v1/v2/v3 |
| `/info` | Show available commands, agents, layout |
| `/memory add <fact>` | (built-in) Persist a global fact in `~/.gemini/GEMINI.md` |
| `/memory refresh` | (built-in) Re-read GEMINI/CONTEXT/MEMORY after editing |
| `/restore` | (built-in) Undo a file change |
| `@<agent>` | Invoke a specialized subagent |

## How session resume works

Gemini hierarchically auto-loads three files into context:

1. `~/.gemini/GEMINI.md` — global rules
2. `<project>/CONTEXT.md` — your project description
3. `<project>/MEMORY.md` — Active threads + Learned facts

Plus `/start` reads the most recent `output/<date>_handoff.md` to see what changed last session.

**Recurrence gate (prevents hallucinated memory):**
- Session 1: agent observes a maybe-fact → stages in handoff's `## Proposed learnings` section
- Session 2: `/start` reads the handoff. If still applicable or user confirms, promotes to MEMORY.md. Otherwise drops it.

After a 2-month gap: `/start` says *"Welcome back. 3 open threads, one stale. 2 facts I observed last session look right — promoting. What are we working on?"*

## Audit & Compliance

State-changing actions write a structured line to `output/audit-log.md`:
```
[2026-04-15 14:23:01] | jsmith | WRITE | output/code/analysis.py | OK
[2026-04-15 14:23:45] | jsmith | QUERY | bigquery:project.dataset.table | 142 rows | OK
```

User identity is captured from the working directory path at session start. Reads, list_directory, and session boundaries are NOT logged.

## Security Defaults

- `.env` files are NEVER read — Gemini reads `.env.example` only
- Project root is OFF LIMITS for new files
- `context/` is READ-ONLY
- Credentials masked as `[REDACTED]` in output
- No PII or raw data in logs — structure only
- No data exfiltration via web fetches without explicit user request
- Each subagent has an explicit `tools:` allowlist (least privilege)
- `PREFERENCES.md` cannot override security rules

## Skills

7 skills shipped in `global/skills/`. Active automatically when the user's task matches.

| Skill | What it knows |
|---|---|
| **excel** | Formulas, dynamic arrays, Power Query M, Power Pivot/DAX, VBA, Office Scripts |
| **python** | pandas/NumPy, scikit-learn, visualization, testing, deployment |
| **bigquery** | Partitioning, clustering, joins, HLL++, cost control, bq CLI |
| **powerbi** | DAX patterns, semantic models, TMDL, XMLA, Tabular Editor, Fabric, CI/CD |
| **tableau-server** | PostgreSQL repository, RMT, jobs, permissions, auditing |
| **tableau-optimizer** | Dashboard performance, TWBX parsing, Power BI migration |
| **tableau-bigquery** | SQL push-down, cost control, BI Engine, materialized views |

## Platform

Built and tested on Windows (PowerShell). The kit also works on macOS/Linux — adjust install commands as shown above.

## License

[MIT](LICENSE)
