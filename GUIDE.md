# Gemini CLI Starter Kit — Guide

## Setup

1. Copy `starter-kit/` contents into your project root.
2. Copy any relevant skills from `examples/skills/` into `.gemini/skills/`.
3. Drop any reference files into `context/` (specs, docs, schemas — auto-loaded every session).
4. Run `/start`. It handles everything — first-time setup and every session after.
5. Run `/help` anytime to see what's available.

Checkpointing is on. Use `/restore` to undo any file change.

## How It Works

**`/start`** is the only command you need. First time, it scans your project, asks smart questions, builds CONTEXT.md, and shows you everything available. Every time after, it loads everything and asks what you're working on. If you left off mid-project, it picks up from your last handoff.

**`/session:save`** ends a session — writes a handoff doc, checks /memory, asks if anything should be permanent.

**`/help`** shows all commands and features anytime.

## What's in the Kit

```
GEMINI.md              Core rules (~65 lines). Imports rules/rules.md.
rules/rules.md         Safety, queries, workflow, logging — all in one file.
CONTEXT.md             Your project context — built by /start, updated by /context:update.
PREFERENCES.md         Your personal preferences — tone, style, habits, whatever you want.
.gemini/settings.json  Checkpointing, compression, session retention, safe tool pre-approvals.
.geminiignore          Keeps noise and secrets out of context.

context/               Reference files — auto-loaded every session.
queries/               SQL query library — one .sql per query, iterated in place.

.gemini/
  commands/            /start, /setup, /session:save, /context:update, /help
  skills/              Skill folders — activated automatically based on your work.

output/
  session-log.md       Task log — auto-appended after every action.
  query-log.md         Query execution log.
```

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume a session |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session — handoff doc + memory check |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/help` | Show available commands and features |
| `/memory add` | Persist a fact across sessions (survives compression) |
| `/restore` | Undo a file change |
| `@./file` | Inject any file into context mid-session |

## Tips

- **context/ is your library.** Drop anything Gemini should always know — auto-loaded, no injection needed.
- **PREFERENCES.md is your personality file.** Want Gemini to be casual? Funny? Terse? Put it there.
- **`/memory add` for critical facts.** Memory survives compression. Chat history doesn't.
- **`@./CONTEXT.md` is your reset button.** If Gemini loses the thread, inject it.
- **`/context:update` before closing.** One minute now saves re-explaining next session.

## Sources

- Official Gemini CLI docs (GEMINI.md, /memory, checkpointing, skills, hooks)
- OWASP AI Agent Security Cheat Sheet (risk classification)
- Google Cloud Community PRAR workflow (gated execution)
- HumanLayer CLAUDE.md research (instruction design)
- AGENTS.md standard (cross-tool compatibility)
