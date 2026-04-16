# Detailed Rules

Core rules (project structure, save-as-you-go, task completion, alignment)
are in GEMINI.md. This file contains the detail layer.

## Safety — action classification

Read-only by default. Classify every action before running it:

  SAFE: read operations, dry runs, previews → proceed
  MEDIUM: creating non-production objects, writing to staging → confirm once
  CRITICAL: any production modification, permission changes, DDL → explicit approval required

- Never modify production unless the user spells out exactly what to change.
- "Clean up the database" is not approval to DROP anything.
- Validate or dry-run before expensive operations when the tool supports it.
- Estimate impact before any data-modifying command.
- Show the full command, report estimated impact, wait for confirmation.

## Query lifecycle

All queries live in ./queries/ as .sql files. One per query, iterated in place.
Include a comment header: purpose, target, date last modified.

Steps (do not skip):
  1. Write it.
  2. Validate (syntax check, dry run if available).
  3. Test it. If wrong, fix and go back to 2.
  4. Show the user. Ask: "Does this look right?"
  5. If no: adjust, back to 2. If yes: save and log.

For simple one-off queries (quick data checks), skip the save — just run, show, log.
The full lifecycle is for queries worth keeping.

Query hygiene:
- No SELECT * — specify columns.
- LIMIT on exploratory queries.
- Always use parameterized queries — never concatenate user input into SQL strings.
- Use cost guardrails if the database supports them.

## Error handling

On failure: show the error, explain in plain language, propose a fix, wait for approval.
Never silently retry. If retrying, vary your approach — don't repeat the same failing attempt.
After two failed attempts, stop and ask.

## Anti-patterns

- NEVER read, search, grep, or open .env files. Ask the user what's configured instead.
- Do not hard-code environment-specific values (IDs, paths, table names).
- Do not take CRITICAL actions without explicit approval.
- Do not produce unrequested output.
- Do not guess. Stop and ask.
- Do not fabricate file contents, API responses, or schema details you haven't verified.
- Do not assume correct retrieval means correct reasoning — verify the logic too.

## Context & memory

- Files in ./context/ auto-load every session.
- /memory add persists facts across sessions.
- /restore reverts file changes (checkpointing enabled).
- Quality degrades as context grows. Compact or start fresh for new task phases.
- For multi-phase work: write findings to files between phases so context can be reset.
- Break complex tasks into discrete steps rather than one long chain.

## Log formats

Session log:  [YYYY-MM-DD HH:MM] <what you did>
Query log:    [YYYY-MM-DD HH:MM] RAN: <name> | Rows: <count> | Result: <summary>

## Session end

/session:save writes a handoff doc and checks memory.
