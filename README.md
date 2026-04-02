# Gemini CLI Starter Kit

A plug-and-play best practices system for [Gemini CLI](https://github.com/google-gemini/gemini-cli). Drop it into any project to get structured session management, persistent memory, automatic logging, safety guardrails, and custom slash commands -- out of the box.

> **New here?** See the [full guide](GUIDE.md) for detailed usage, tips, and command reference.

## What You Get

- **Gated execution** -- Gemini follows an Explain > Plan > Implement workflow for every non-trivial task
- **Risk classification** -- every action is classified (Low / Medium / High / Critical) before execution
- **Automatic logging** -- prompts, tasks, data commands, learnings, and decisions are logged without you asking
- **Session management** -- structured start (`/init`, `/brief`) and end (`/session:save`) with handoff docs
- **Custom slash commands** -- `/setup`, `/init`, `/brief`, `/session:save`, `/context:update`
- **Security defaults** -- credentials never in output, checkpointing enabled
- **Output organization** -- clean folder structure for code, data, reports, notes, and scratch files

## Quick Start

### 1. Copy the kit into your project

Clone this repo (or download the zip), then copy the contents of `starter-kit/` into your project root:

```
starter-kit/    <-- copy everything inside this folder into your project
```

That's it -- just the contents of `starter-kit/`, nothing else from the repo.

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

Everything else runs automatically.

## What's in `starter-kit/`

```
GEMINI.md                  # Core rules -- auto-loaded every session
CONTEXT.md                 # Your project context -- fill via /setup, update via /context:update
settings.json              # Checkpointing, session retention, compression settings
.geminiignore              # Keeps secrets and noise out of context

context/                   # Drop reference files here -- auto-loaded every session
queries/                   # SQL query library -- one .sql per query, iterated in place

.gemini/
  commands/
    setup.toml             # /setup -- first-time context builder
    init.toml              # /init -- full session start
    brief.toml             # /brief -- fast session start
    session/save.toml      # /session:save -- session end + handoff
    learnings/add.toml     # /learnings:add -- log a discovery immediately
    context/update.toml    # /context:update -- add to CONTEXT.md permanently
  skills/                  # Drop skill folders here (each with a SKILL.md)

output/
  session-log.md           # Lightweight session log
  query-log.md             # Query execution log (what ran, rows, bytes, cost)
```

## Commands

| Command | What it does |
|---|---|
| `/setup` | First-time only -- builds `CONTEXT.md` via interview |
| `/init` | Full session start with intake questions |
| `/brief` | Fast session start, one question |
| `/session:save` | Session end -- writes handoff doc, checks memory |
| `/context:update` | Add something permanent to `CONTEXT.md` |
| `/stats` | Check context window usage |
| `/memory add <fact>` | Persist a fact across sessions |
| `/skills list` | View all discovered skills |
| `@./filename` | Inject any file into context mid-session |

## What Runs Automatically

- Tasks logged to `output/session-log.md`
- SQL queries saved to `queries/`, tested before finalizing, executions logged to `output/query-log.md`
- Diff shown before any existing file is modified
- Dry run used before data commands when available
- Bulk operations require itemized list + explicit approval
- Commands shown and confirmed before execution
- Failures explained in plain language with proposed fix

## Skills

Gemini CLI has a native skills system. A skill is a folder with a `SKILL.md` file that teaches Gemini specialized knowledge. Drop skill folders into `.gemini/skills/` and Gemini activates them when relevant.

The `examples/skills/` folder in this repo contains sample skills you can copy into your project:

- **`tableau-bigquery/`** -- Tableau + BigQuery live connection expertise (SQL push-down, cost control, auth, monitoring)

To install a skill:

```
cp -r examples/skills/tableau-bigquery your-project/.gemini/skills/
```

To create your own, add a folder with a `SKILL.md` to `.gemini/skills/`. See the [Gemini CLI skills docs](https://geminicli.com/docs/cli/skills/) for details.

## Based On

- Official Gemini CLI documentation (GEMINI.md system, /memory, checkpointing)
- OWASP AI Agent Security Cheat Sheet (risk-tiered action classification)
- HumanLayer CLAUDE.md research (instruction design patterns)
- Google Cloud Community PRAR workflow (gated execution)
- AGENTS.md standard (cross-tool compatibility)
- addyosmani/gemini-cli-tips (compression-aware memory management)

See [GUIDE.md](GUIDE.md) for the full list of sources.

## License

[MIT](LICENSE)
