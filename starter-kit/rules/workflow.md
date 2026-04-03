# Workflow & Session Management

## Task completion definition

A task is NOT complete until all of these are true:
  1. The work is done.
  2. If it produces output (code, queries, scripts): you tested it and the user confirmed it.
  3. It is logged in ./output/session-log.md.
  4. If a query was run, it is logged in ./output/query-log.md.

Logging is not a separate step — it is part of finishing the task.
Do not respond to the user until the log entry is written.

Session log format:   [YYYY-MM-DD HH:MM] <what you did>
Query log format:     [YYYY-MM-DD HH:MM] RAN: <filename or description> | Rows: <count> | Result: <summary>

## Alignment Checks

Before any multi-step or non-trivial task:
  → Restate what you're about to do in one sentence. Wait for confirmation.

After completing a task:
  → Briefly confirm what was done. Ask if it meets expectations.

If the user corrects you:
  → Acknowledge the correction. Adjust immediately. Do not repeat the mistake.

If you're about to do something that could go wrong or be hard to undo:
  → Say what you're about to do and why. Ask before proceeding.

## Output & Files

- One file per deliverable. Iterate in place — no copies or versions.
- Queries go in ./queries/. Everything else in ./output/.
- Use clear, descriptive filenames.
- Delete anything no longer needed. Keep it clean.

## Context & Memory

- Files in ./context/ are auto-loaded every session.
- /memory add persists facts across sessions (survives compression).
- /memory show to see what's persisted.
- /stats to check context window usage — persist critical facts before compression.
- Use /restore or double-Esc to revert file changes (checkpointing enabled).

## Session End

Run /session:save to write a handoff doc:
  - What we did
  - What's open/unfinished
  - Key learnings
  - What to carry forward

Before closing, it asks if anything should go into CONTEXT.md permanently.
