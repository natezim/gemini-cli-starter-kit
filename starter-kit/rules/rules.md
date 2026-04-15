# Rules

## Save As You Go

Write to disk IMMEDIATELY after every meaningful action. Do not wait.

  → ./output/session-log.md — append what you just did
  → ./output/query-log.md — append if you ran a query
  → ./output/<descriptive-name> — write findings, results, analysis to files
  → /memory add — persist critical discoveries and decisions

If you produced findings and they only exist in conversation, STOP and write them now.
Context can be lost at any time. Everything not on disk is at risk.

Session log:  [YYYY-MM-DD HH:MM] <what you did>
Query log:    [YYYY-MM-DD HH:MM] RAN: <name> | Rows: <count> | Result: <summary>

## Task Completion

A task is NOT complete until:
  1. Results are written to disk (not just in conversation).
  2. If it produces output: you tested it and the user confirmed it.
  3. It is logged.

## Alignment

Before multi-step or non-trivial tasks: restate what you're about to do. Confirm.
After completing a task: confirm what was done. Ask if it meets expectations.
If corrected: acknowledge, adjust, don't repeat.
If something could go wrong or is hard to undo: say so and ask first.

## Safety

Read-only by default. Classify every action:

  SAFE: read operations, dry runs, previews → proceed
  MEDIUM: creating non-production objects, writing to staging → confirm once
  CRITICAL: any production modification, permission changes, DDL → explicit approval required

- Never modify production unless the user spells out exactly what to change.
- "Clean up the database" is not approval to DROP anything.
- Validate or dry-run before expensive operations when the tool supports it.
- Estimate impact before any data-modifying command.
- Show the full command, report estimated impact, wait for confirmation.

## Queries

All queries live in ./queries/ as .sql files. One per query, iterated in place.
Include a comment header: purpose, target, date last modified.

Lifecycle:
  1. Write it.
  2. Validate (syntax check, dry run if available).
  3. Test it. If wrong, fix and go back to 2.
  4. Show the user. Ask: "Does this look right?"
  5. If no: adjust, back to 2. If yes: save and log.

For simple one-off queries (quick data checks), skip the save — just run, show, log.
The full lifecycle is for queries worth keeping.

Hygiene:
- No SELECT * — specify columns.
- LIMIT on exploratory queries.
- Use cost guardrails if the database supports them.

## Output & Files

One file per deliverable. Iterate in place. Delete what's no longer needed.

File locations — never dump at project root:
  → SQL queries (.sql) → ./queries/
  → Scripts, code (.py, .js, .sh) → ./output/code/
  → Reports, analysis (.md) → ./output/reports/
  → Data exports (.csv, .json, .xlsx) → ./output/data/
  → Plans → .gemini/plans/ (plan mode only)
  → Logs → ./output/session-log.md, ./output/query-log.md

Create the target folder if it doesn't exist. Ask the user if unsure.

## Error Handling

On failure: show the error, explain in plain language, propose a fix, wait for approval.
Never silently retry. After two failed attempts, stop and ask.

## Anti-Patterns

- Do not silently retry. If retrying, vary your approach — don't repeat the same failing attempt.
- Do not hard-code environment-specific values.
- Do not take CRITICAL actions without explicit approval.
- Do not produce unrequested output.
- Do not guess. Stop and ask.
- Do not fabricate file contents, API responses, or schema details you haven't verified.
- Do not assume correct retrieval means correct reasoning — verify the logic too.

## Context & Memory

- Files in ./context/ auto-load every session.
- /memory add persists facts across sessions.
- /restore reverts file changes (checkpointing enabled).
- Quality degrades as context grows. Compact or start fresh sessions for new task phases.
- For multi-phase work: write findings to files between phases so context can be reset.
- Break complex tasks into discrete steps rather than one long chain.

## Session End

/session:save writes a handoff doc and checks memory.
