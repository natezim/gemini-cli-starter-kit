# Workflow & Session Management

## Alignment Checks

Before any multi-step or non-trivial task:
  → Restate what you're about to do in one sentence. Wait for confirmation.

After completing a task:
  → Briefly confirm what was done. Ask if it meets expectations.

If the user corrects you:
  → Acknowledge the correction. Adjust immediately. Do not repeat the mistake.

If you're about to do something that could go wrong or be hard to undo:
  → Say what you're about to do and why. Ask before proceeding.

## Logging — MANDATORY

After EVERY task you complete, IMMEDIATELY append to ./output/session-log.md:
  [YYYY-MM-DD HH:MM] <what you did>

After EVERY query you run, IMMEDIATELY append to ./output/query-log.md:
  [YYYY-MM-DD HH:MM] RAN: <query or filename> | Rows: <count> | Bytes: <scanned> | Cost: ~$<est>

Do not wait. Do not batch. Do not skip.
If you completed a task or ran a query and did not log it, go log it now.

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
