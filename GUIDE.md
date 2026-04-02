# Gemini CLI — Best Practices Starter Kit
# Getting Started Guide

This kit gives Gemini CLI structured session management, persistent memory,
automatic logging, and safety guardrails out of the box — for any project,
any tool, any user. Drop it in, run /setup, and go.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## WHAT'S IN THE KIT

  GEMINI.md              Base rules. Auto-loaded every session. Customize as needed.
  CONTEXT.md             Your project context. Fill once, update often.
  SESSION.md             Created fresh each session. Tracks today's goal.
  settings.json          Enables checkpointing and security settings.
  .geminiignore          Keeps noise and secrets out of context.

  .gemini/commands/
    /setup               First-time wizard — builds CONTEXT.md via interview
    /init                Full session start with intake interview
    /brief               Fast session start, one question
    /session:save        Session end — handoff doc + memory check
    /learnings:add       Log a discovery immediately
    /context:update      Add something permanently to CONTEXT.md

  output/                Everything Gemini writes goes here
    prompt-log.md        Every prompt, auto-logged
    session-log.md       Every task, auto-logged with risk level
    code/                Finalized scripts and queries
    data/                Exports and analysis
    reports/             Summaries and reports
    scratch/             Temp files — always cleaned up
    notes/
      learnings.md       Discoveries, logged as they happen
      decisions.md       Decisions made, running log
      data-changelog.md  Every query/command, before and after
      YYYY-MM-DD_handoff.md   Session wrap-up docs

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## FIRST-TIME SETUP (do this once)

1. Copy the entire kit into your project folder.
   Keep the folder structure exactly as-is.

2. Run /setup
   Gemini interviews you — 10 questions, one at a time.
   It writes CONTEXT.md with your actual answers. No placeholders.
   Takes about 5 minutes. The better your context, the better every session.

3. Done.
   settings.json enables checkpointing automatically.
   You can undo any file change with /restore or double-Esc.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## STARTING A SESSION

  /init    Full onboarding. Use this when starting something new, coming back
           after time away, or when you want a structured documented session.

           Gemini will:
           → Read CONTEXT.md and summarize your environment
           → Check the last 3 session log entries
           → Read the last 5 learnings
           → Surface anything persisted from past sessions via /memory show
           → Ask 4 intake questions (goal, scope, constraints, definition of done)
           → Write SESSION.md
           → Say "Ready. What's the first task?"

  /brief   Fast start. Use for routine sessions when you know what you're doing.

           Gemini asks one question:
           "Quick brief — what are we doing today and anything I need to know?"
           Reads CONTEXT.md silently, writes SESSION.md, ready in under 10 seconds.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## DURING A SESSION

Re-inject context at any time with @
If Gemini loses the thread mid-session, inject any file directly:

  @./CONTEXT.md                      Re-inject project context
  @./output/notes/learnings.md       Re-inject accumulated learnings
  @./output/notes/data-changelog.md  Review recent command history
  @./output/scratch/test_query       Share a draft query or script

Log a discovery immediately
When something worth remembering comes up:

  /learnings:add

Don't wait until session end. Things get compressed away.

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
→ Write a full handoff doc to output/notes/YYYY-MM-DD_handoff.md
→ Check /memory show and persist anything critical not yet saved
→ Ask if anything should be added to CONTEXT.md permanently
→ Confirm everything is closed out

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## COMMAND REFERENCE

  /setup            First-time only — builds CONTEXT.md via interview
  /init             Full session start with intake
  /brief            Fast session start, one question
  /session:save     Session end, handoff doc, memory check
  /learnings:add    Log a discovery to learnings.md right now
  /context:update   Add something permanent to CONTEXT.md
  /stats            Check context window usage
  /memory add       Persist a fact across sessions
  /memory show      See facts persisted from past sessions
  /restore          Undo a file change (requires checkpointing)
  @./filename       Inject any file into context mid-session

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## WHAT RUNS AUTOMATICALLY

You never have to ask for any of these:

  ✓ Every prompt logged verbatim to output/prompt-log.md
  ✓ Every task logged with risk level to output/session-log.md
  ✓ Every data command logged with before/after to output/notes/data-changelog.md
  ✓ Diff shown before any existing file is modified
  ✓ Dry run used before any data command if the tool supports it
  ✓ Bulk operations require an itemized list + explicit yes
  ✓ Every command shown and confirmed before execution
  ✓ Scratch files cleaned up at session end
  ✓ Failures logged with error message and plain-language explanation

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## TIPS

/setup is worth doing properly.
The more you put into CONTEXT.md upfront, the less you explain every session.
Include recurring tasks, known gotchas, naming conventions, and safety limits.

/learnings:add in the moment beats waiting.
Context gets compressed during long sessions. Log discoveries immediately.

@./CONTEXT.md is your reset button.
If Gemini loses the thread, inject it and everything reloads mid-session.

/memory add for the most critical facts.
learnings.md is a file — it has to be re-read each session.
/memory entries are natively persistent and survive compression automatically.

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
