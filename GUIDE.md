# Gemini CLI Starter Kit — Guide

## Setup

1. Copy `starter-kit/` contents into your project root.
2. Drop any reference files into `context/` (specs, docs, schemas — auto-loaded every session).
3. Run `/init`. It handles everything — first-time setup and every session after.

That's it. Checkpointing is on. Use `/restore` or double-Esc to undo any file change.

## How It Works

**`/init`** is the only command you need to start. First time, it scans your project (context files, .env variable names, saved queries), asks a few questions to fill gaps, and writes CONTEXT.md. Every time after, it loads everything and asks what you're working on.

**`/session:save`** ends a session — writes a handoff doc, checks /memory, and asks if anything should be permanent in CONTEXT.md.

## What's in the Kit

```
GEMINI.md              Core rules (~70 lines). Imports from rules/.
rules/
  safety.md            Action classification, production guardrails, anti-patterns
  sql.md               Query library, testing, hygiene, execution logging
  workflow.md           Logging, output, session management, context & memory
CONTEXT.md             Your project context — built by /init, updated by /context:update
settings.json          Checkpointing, compression, session retention, safe tool pre-approvals
.geminiignore          Keeps secrets and noise out of context

context/               Reference files — auto-loaded every session
queries/               SQL query library — one .sql per query, iterated in place

.gemini/
  commands/            Slash commands (/init, /setup, /session:save, /context:update)
  skills/              Skill folders — each with a SKILL.md, activated when relevant

output/
  session-log.md       Task log — auto-appended after every action
  query-log.md         Query execution log — what ran, rows, bytes, cost
```

## Commands

| Command | What it does |
|---|---|
| `/init` | Start a session (first-time setup or returning) |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session — handoff doc + memory check |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/stats` | Check context window usage |
| `/memory add` | Persist a fact across sessions (survives compression) |
| `/skills list` | View discovered skills |
| `/restore` | Undo a file change |
| `@./file` | Inject any file into context mid-session |

## Tips

- **context/ is your library.** Drop anything Gemini should always know — auto-loaded, no injection needed.
- **`/memory add` for critical facts.** Memory survives compression. Chat history doesn't.
- **`@./CONTEXT.md` is your reset button.** If Gemini loses the thread, inject it.
- **`/context:update` before closing.** One minute now saves re-explaining next session.

## What Runs Automatically

- Tasks logged after every action
- Queries tested before saving, executions logged with cost
- Dry run before expensive operations
- Diffs shown before modifying existing files
- No `SELECT *` — columns must be specified
- Bulk operations require itemized approval
- Failures explained with proposed fix

## Sources

- Official Gemini CLI docs (GEMINI.md, /memory, checkpointing, skills, hooks)
- OWASP AI Agent Security Cheat Sheet (risk classification)
- Google Cloud Community PRAR workflow (gated execution)
- HumanLayer CLAUDE.md research (instruction design)
- AGENTS.md standard (cross-tool compatibility)
- addyosmani/gemini-cli-tips (compression-aware memory)
