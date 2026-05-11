# Gemini CLI Starter Kit — Guide

Built for data analysts working with SQL, SAS, Python, BigQuery, Excel, Power Query, Power BI, and Tableau. **13 specialized subagents**, smart session-resume, query management, production safety, deep skills — out of the box.

## Setup

### 1. Global install (once per machine)

```powershell
Copy-Item -Recurse global\* "$env:USERPROFILE\.gemini\"
```

```bash
cp -r global/* ~/.gemini/
```

This installs everything that's the same for every project:
- `GEMINI.md` (rules), `PREFERENCES.md`, `settings.json`
- All 13 custom subagents
- All 7 skills
- Custom slash commands (`/start`, `/session:save`, `/version`, `/info`)

### 2. Per-project setup

For each project where you want the kit active:

```powershell
Copy-Item -Recurse <kit-repo>\project-template\* .
```

You'll get `CONTEXT.md`, `MEMORY.md`, sensible `.gitignore`/`.geminiignore`, `.env.example`, and an `output/` workspace.

Then run `/start` inside Gemini CLI. First time, it scans your project, asks only what it can't infer, and writes `CONTEXT.md`. After that, every `/start` resumes from your last session.

### 3. Updates

Pull the latest from this repo, copy `global/*` into `~/.gemini/` again. All projects pick up the new rules on next `/start`. Project files (`CONTEXT.md`, `MEMORY.md`, `output/`) are untouched.

## How it works

### `/start` — start or resume

Captures your user identity, snapshots the file list (for end-of-session versioning), and either runs first-time setup or resumes.

**Returning session:** reads the most recent `output/<date>_handoff.md`. If the handoff has `## Proposed learnings`, agent decides which ones to promote to `MEMORY.md` based on whether they still apply or you confirm them. Highlights any stale `Active threads` (>30 days untouched).

After a long gap, `/start` says something like:
> "Welcome back. Project: <one-line>. 3 open threads — one is stale (>60 days). Promoted 2 proposed learnings to memory. What are we working on?"

### Skills

When your task matches a skill, it activates automatically. No invocation needed.

### Subagents

Type `@` in chat — picker shows all 13 agents + 3 built-ins. Each runs in an isolated context window so heavy work doesn't bloat the main session. See [AGENTS.md](AGENTS.md) for the decision tree.

### Plan Mode

On for multi-step or risky work. Gemini researches first (Pro), proposes a plan, waits for your approval, then implements (Flash). Trivial requests skip planning automatically.

### Queries

Write → validate (`@query-validator`) → save → log. Library lives in `output/queries/`.

### File discipline

| Type | Goes in |
|---|---|
| `.sql` | `output/queries/` |
| `.py`, `.js`, `.sh`, configs | `output/code/` |
| `.md` (reports) | `output/reports/` |
| `.csv`, `.json`, `.xlsx`, `.parquet` | `output/data/` |
| Anything throwaway | `output/temp/` (auto-wiped at session end) |

No scratch sprawl. No files at project root. No `_v2`/`_final`/`_new` suffixes — versioning happens at `/session:save`.

### Memory — 3 tiers

| File | Owner | What |
|---|---|---|
| `~/.gemini/GEMINI.md` | You (via `/memory add`) | Global cross-project preferences |
| `<project>/CONTEXT.md` | You | Project description, conventions, schemas |
| `<project>/MEMORY.md` | Agent (you prune) | `## Active threads` + `## Learned` |

`MEMORY.md` is auto-loaded every session. Edit `CONTEXT.md` and run `/memory refresh` to apply changes mid-session.

**Recurrence gate** prevents fabricated learnings: when the agent observes a maybe-fact, it stages the entry in the handoff doc's `## Proposed learnings`. Next `/start` promotes it to `MEMORY.md` only if the fact still holds or you confirm it. Single-occurrence speculation never makes it to permanent memory.

### Audit log

Silent. Only fires on state changes:
```
[2026-04-15 14:23:01] | jsmith | WRITE | output/code/analysis.py | OK
[2026-04-15 14:23:45] | jsmith | QUERY | bigquery:project.dataset.table | 142 rows | OK
```

Reads, `list_directory`, and session boundaries are NOT logged.

### Versioning

At `/session:save`, modified files in `output/code/`, `output/reports/`, `output/queries/` get snapshotted as `_v1`, `_v2`, `_v3` (max 3, rotating). The live working file keeps its base name.

### Handoff doc

Per-session diff (not a snapshot of the whole project). Sections:
- What changed this session
- What failed (verbatim — no smoothing over)
- Open / unfinished
- Proposed learnings (staged for next session's promotion)
- Next session should…

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume |
| `/session:save` | End session — versioning, MEMORY.md update, handoff |
| `/version <file>` | Manual snapshot to v1/v2/v3 |
| `/info` | Show available commands, agents, layout |
| `/memory add <fact>` | (built-in) Persist a global fact in `~/.gemini/GEMINI.md` |
| `/memory refresh` | (built-in) Re-load GEMINI/CONTEXT/MEMORY after editing |
| `/restore` | (built-in) Undo a file change |
| `@<agent>` | Invoke a specialized subagent |
| `@./file` | Inject any file into context mid-session |

## Subagents — Quick Reference

See [AGENTS.md](AGENTS.md) for the decision tree with example prompts.

**Thinking:** `@solution-designer`

**SQL / data:** `@query-validator`, `@schema-explorer`, `@data-profiler`

**Stats / migration:** `@stats-advisor`, `@sas-migrator`

**Excel / Power Query / DAX:** `@excel-reviewer`, `@powerquery-reviewer`, `@dax-reviewer`

**BI tools:** `@tableau-auditor`, `@viz-optimizer`, `@migration-mapper`

**Output:** `@report-drafter`

## Tips

- **`context/` is your project library.** Drop schemas, docs, specs — auto-loaded, READ-ONLY.
- **`output/queries/` is your SQL library.** One file per query, iterated in place, every execution logged.
- **`PREFERENCES.md` is for tone, agent defaults, workflow overrides.**
- **Stuck on a vague problem? `@solution-designer` first.** Turns "build a dashboard" into a concrete plan.
- **Validate before executing on billable systems.** `@query-validator` does dry-run cost checks before BigQuery scans.
- **Don't fight versioning.** Iterate in place; versioning happens at `/session:save`.

## Sources

- Official Gemini CLI docs (GEMINI.md hierarchy, `/memory`, checkpointing, skills, subagents)
- OWASP AI Agent Security Cheat Sheet (risk classification)
- Google Cloud Community PRAR workflow (gated execution)
- HumanLayer CLAUDE.md research (instruction design)
- AGENTS.md standard (cross-tool compatibility)
- LLM limitations research (context management, fabrication prevention, recurrence gating)
- Deep research on Gemini subagent architecture (May 2026)
