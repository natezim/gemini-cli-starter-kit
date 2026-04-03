# Gemini CLI — Best Practices Starter Kit
# Getting Started Guide

This kit gives Gemini CLI structured session management, persistent memory,
automatic logging, and safety guardrails out of the box — for any project,
any tool, any user. Drop it in, run /setup, and go.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## WHAT'S IN THE KIT

  GEMINI.md              Core rules (~70 lines). Imports detailed rules from rules/.
  rules/
    safety.md            Data & command safety, action classification, anti-patterns.
    sql.md               Query library, testing, hygiene, execution logging.
    workflow.md           Session management, logging, output, context & memory.
  CONTEXT.md             Your project context. Fill once, update often.
  SESSION.md             Created fresh each session. Tracks today's goal.
  settings.json          Checkpointing, session retention, compression, safe tools.
  .geminiignore          Keeps noise and secrets out of context.

  context/               Drop reference files here — auto-loaded every session.
  queries/               SQL query library — one .sql per query, iterated in place.

  .gemini/
    commands/
      /init              Start a session — handles first-time setup + returning
      /setup             Rebuild context from scratch
      /session:save      Session end — handoff doc + memory check
      /context:update    Add something permanently to CONTEXT.md
    skills/              Drop skill folders here — each with a SKILL.md

  output/                Deliverables and logs
    session-log.md       Lightweight session log, auto-appended
    query-log.md         Query execution log — what ran, rows, bytes, cost

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## FIRST-TIME SETUP (do this once)

1. Copy the contents of starter-kit/ into your project root.
   Keep the folder structure exactly as-is.

2. Drop any reference files into ./context/
   Specs, docs, schemas, API references, examples — anything Gemini should know.
   These are auto-loaded every session and /setup will use them to tailor its questions.

3. Run /setup
   Gemini interviews you — 10 questions, one at a time.
   It writes CONTEXT.md with your actual answers. No placeholders.
   Takes about 5 minutes. The better your context, the better every session.

4. Done.
   settings.json enables checkpointing automatically.
   You can undo any file change with /restore or double-Esc.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## STARTING A SESSION

  /init    The only command you need. Handles everything.

           First time? Scans your project, asks smart questions, builds CONTEXT.md.
           Returning? Loads context, shows status, asks what you're doing. Ready.

           Use /setup only if you want to rebuild your context from scratch.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## DURING A SESSION

Add reference files to context/
Drop specs, docs, schemas, or examples into ./context/ and they'll be
auto-loaded every session. No @ injection needed for persistent files.

Add skills for specialized expertise
Drop a skill folder (containing a SKILL.md) into .gemini/skills/.
Gemini discovers skills at session start and activates them when relevant.
Use /skills list to see available skills and /skills reload after adding new ones.

Re-inject context at any time with @
If Gemini loses the thread mid-session, inject any file directly:

  @./CONTEXT.md                      Re-inject project context
  @./output/session-log.md           Review session history
  @./output/my-query.sql             Share a file inline

Add something permanently to your context
When a discovery should follow you into every future session:

  /context:update

Check context window usage
  /stats

When usage hits ~40%, persist critical facts:
  /memory add <the fact>

Memory persists natively across sessions.
Chat history gets compressed. /memory does not.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## ENDING A SESSION

  /session:save

Gemini will:
→ Write a handoff doc to output/YYYY-MM-DD_handoff.md
→ Check /memory show and persist anything critical not yet saved
→ Ask if anything should be added to CONTEXT.md permanently
→ Confirm everything is closed out

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## COMMAND REFERENCE

  /init             Start a session — first-time setup or returning
  /setup            Rebuild context from scratch
  /session:save     Session end, handoff doc, memory check
  /context:update   Add something permanent to CONTEXT.md
  /stats            Check context window usage
  /memory add       Persist a fact across sessions
  /memory show      See facts persisted from past sessions
  /skills list      View all discovered skills
  /skills reload    Reload skills after adding new ones
  /restore          Undo a file change (requires checkpointing)
  @./filename       Inject any file into context mid-session

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## WHAT RUNS AUTOMATICALLY

You never have to ask for any of these:

  ✓ Tasks logged to output/session-log.md
  ✓ SQL queries saved to queries/, tested before finalizing, executions logged
  ✓ Diff shown before any existing file is modified
  ✓ Dry run used before any data command if the tool supports it
  ✓ Bulk operations require an itemized list + explicit yes
  ✓ Every command shown and confirmed before execution
  ✓ Files iterated in place — one final copy, no clutter
  ✓ Failures explained in plain language with proposed fix

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## TIPS

The context/ folder is your library.
Drop any file Gemini should always know about — API specs, schema docs, style
guides, example outputs. They're loaded automatically every session.

/setup is worth doing properly.
The more you put into CONTEXT.md upfront, the less you explain every session.
Include recurring tasks, known gotchas, naming conventions, and safety limits.

@./CONTEXT.md is your reset button.
If Gemini loses the thread, inject it and everything reloads mid-session.

/memory add for the most critical facts.
Memory entries are natively persistent and survive compression automatically.
Use it for gotchas, key decisions, and anything you'd hate to re-explain.

/context:update before closing.
Before every session ends, ask: did we learn anything that should be permanent?
One minute now saves time in every future session.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## BASED ON

- Official Gemini CLI documentation (GEMINI.md system, /memory, checkpointing, hooks)
- OWASP AI Agent Security Cheat Sheet (risk-tiered action classification)
- Meta Rule of Two (agent permission scoping)
- HumanLayer CLAUDE.md research (instruction design, negative examples, brevity)
- Google Cloud Community PRAR workflow (gated execution)
- AGENTS.md standard / Linux Foundation (cross-tool compatibility)
- Arcade.dev SQL agent guide (data command safety)
- addyosmani/gemini-cli-tips (compression-aware memory management)
- Prashanth Subrahmanyam / Google Cloud (structured GEMINI.md approach)
