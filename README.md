# Gemini CLI Starter Kit

A plug-and-play best practices system for [Gemini CLI](https://github.com/google-gemini/gemini-cli). Drop it into any project to get structured session management, persistent memory, automatic logging, safety guardrails, and custom slash commands -- out of the box.

## What You Get

- **Gated execution** -- Gemini follows an Explain > Plan > Implement workflow for every non-trivial task
- **Risk classification** -- every action is classified (Low / Medium / High / Critical) before execution
- **Automatic logging** -- prompts, tasks, data commands, learnings, and decisions are logged without you asking
- **Session management** -- structured start (`/init`, `/brief`) and end (`/session:save`) with handoff docs
- **Custom slash commands** -- `/setup`, `/init`, `/brief`, `/session:save`, `/learnings:add`, `/context:update`
- **Security defaults** -- credentials never in output, checkpointing enabled, YOLO mode disabled
- **Output organization** -- clean folder structure for code, data, reports, notes, and scratch files

## Quick Start

### 1. Clone into your project

```bash
git clone https://github.com/natezim/gemini-cli-starter-kit.git
cp -r gemini-cli-starter-kit/* gemini-cli-starter-kit/.gemini gemini-cli-starter-kit/.geminiignore your-project/
```

Or copy the files manually into your project root.

### 2. Run first-time setup

```
/setup
```

Gemini interviews you with 10 questions (one at a time) and writes `CONTEXT.md` with your actual answers. Takes about 5 minutes. The better your context, the better every session.

### 3. Start a session

```
/init     # Full onboarding -- loads context, checks history, runs intake interview
/brief    # Fast start -- one question, ready in seconds
```

That's it. Everything else runs automatically.

## File Structure

```
GEMINI.md                  # Core rules -- auto-loaded every session, don't edit
CONTEXT.md                 # Your project context -- fill via /setup, update via /context:update
settings.json              # Checkpointing + security settings
.geminiignore              # Keeps secrets and noise out of context

.gemini/commands/
  setup.toml               # /setup -- first-time context builder
  init.toml                # /init -- full session start
  brief.toml               # /brief -- fast session start
  session-save.toml        # /session:save -- session end + handoff
  learnings-add.toml       # /learnings:add -- log a discovery immediately
  context-update.toml      # /context:update -- add to CONTEXT.md permanently

output/
  prompt-log.md            # Every prompt, auto-logged
  session-log.md           # Every task + risk level, auto-logged
  code/                    # Finalized scripts and queries
  data/                    # Exports, CSVs, analysis results
  reports/                 # Summaries and reports
  scratch/                 # Temp files -- always cleaned up
  notes/
    learnings.md           # Discoveries, logged as they happen
    decisions.md           # Decision log
    data-changelog.md      # Every query/command with before/after
```

## Commands

| Command | What it does |
|---|---|
| `/setup` | First-time only -- builds `CONTEXT.md` via interview |
| `/init` | Full session start with intake questions |
| `/brief` | Fast session start, one question |
| `/session:save` | Session end -- writes handoff doc, checks memory |
| `/learnings:add` | Log a discovery to `learnings.md` right now |
| `/context:update` | Add something permanent to `CONTEXT.md` |
| `/stats` | Check context window usage |
| `/memory add <fact>` | Persist a fact across sessions |
| `@./filename` | Inject any file into context mid-session |

## What Runs Automatically

- Every prompt logged to `output/prompt-log.md`
- Every task logged with risk level to `output/session-log.md`
- Every data command logged with before/after diff
- Diff shown before any existing file is modified
- Dry run used before data commands when available
- Bulk operations require itemized list + explicit approval
- Scratch files cleaned up at session end
- Failures logged with error message and plain-language explanation

## Based On

- [Gemini CLI docs](https://github.com/google-gemini/gemini-cli) (GEMINI.md system, /memory, checkpointing)
- [OWASP AI Agent Security Cheat Sheet](https://owasp.org/www-project-ai-security/) (risk-tiered action classification)
- [HumanLayer CLAUDE.md research](https://humanlayer.dev/) (instruction design patterns)
- [Google Cloud Community PRAR workflow](https://cloud.google.com/) (gated execution)
- [AGENTS.md standard](https://agents-md.org/) (cross-tool compatibility)
- [addyosmani/gemini-cli-tips](https://github.com/nichochar/open-source-agentic-coding-book/blob/main/src/tips-and-tricks.md) (compression-aware memory management)

## License

MIT
