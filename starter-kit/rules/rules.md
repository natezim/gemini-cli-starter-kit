# Detailed Rules

Core rules (file discipline, save-as-you-go, audit log, alignment) are in GEMINI.md.
This file is the detail layer.

## Audit log format

`./output/audit-log.md` — append-only, structured, one line per action:

```
[YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>
```

User identity: parse from working directory at session start (e.g., `C:\Users\jsmith\...` → `jsmith`).
Fallback: `%USERNAME%` env var. Set once, used in every entry.

Action codes:
- `WRITE` — created a new file
- `EDIT` — modified an existing file
- `DELETE` — removed a file
- `EXEC` — ran a script or shell command
- `QUERY` — ran a database query
- `READ-EXT` — read content from outside the project (web fetch, etc.)

Result: `OK`, `FAIL`, or short context (e.g., `142 rows`, `exit 0`, `2.1MB scanned`).
Never log file contents, query results data, or credentials — structure only.

## Versioning — automatic at session end

At session end (`/session:save`), versions are managed automatically for files in:
`./output/code/`, `./output/reports/`, `./queries/`.

Logic:
1. Compare current files to the snapshot taken at session start.
2. For files that EXISTED at session start AND were modified this session:
   - Rotate existing versions: `_v3` is dropped, `_v2` → `_v3`, `_v1` → `_v2`.
   - Snapshot current to `<name>_v1.<ext>`.
3. The live working file keeps its base name.

Result: max 3 historical versions per file. Live file is always the latest.

For mid-session snapshots, use `/version <filename>` (manual escape hatch).

## Cleanup review at session end

At `/session:save`:
1. Compare current files to the snapshot from session start.
2. List files that are NEW this session (didn't exist at start).
3. Ask user: "Keep all, delete all, or review one-by-one?"
4. Default action if user just confirms: keep all.
5. Apply user's choice.

No mid-session interruptions for cleanup. Ask once at the end.

## Chat log — substantive prompts only

At `/session:save`, save user prompts to `./output/prompts/<date>_prompts.md`.

Filter rule: SKIP a prompt if BOTH conditions are true:
- Under 5 words.
- Matches confirmation pattern: yes, ok, okay, sure, go, go ahead, proceed, continue,
  do it, do that, fine, good, looks good, sounds good, yep, yeah, k, thanks, thank you.

Capture everything else verbatim with timestamp:

```markdown
# Prompts — 2026-04-15 — jsmith

[14:20:33] What's the structure of this database?
[14:31:05] Run a query to find duplicate emails over the last 30 days
[14:45:12] Save that to a SQL file in queries/
```

## Session start snapshot

At `/start`, capture:
1. User identity from working directory path.
2. List of all files in `./output/code/`, `./output/reports/`, `./queries/`.
3. Write opening audit entry with user + timestamp.

This snapshot is used by `/session:save` to detect new vs modified files.

## Safety — action classification

Read-only by default. Classify every action before running it:

  SAFE: read operations, dry runs, previews → proceed
  MEDIUM: creating non-production objects, writing to staging → confirm once
  CRITICAL: production modification, permission changes, DDL → explicit approval

- Never modify production unless the user spells out exactly what to change.
- "Clean up the database" is not approval to DROP anything.
- Validate or dry-run before expensive operations when supported.
- Show the full command, report estimated impact, wait for confirmation.

## Query lifecycle

All queries live in `./queries/` as `.sql` files. One per query, iterated in place.
Include a comment header: purpose, target, date last modified.

Steps (do not skip):
  1. Write it.
  2. Validate (syntax check, dry run if available).
  3. Test it. If wrong, fix and go back to 2.
  4. Show the user. Ask: "Does this look right?"
  5. If no: adjust, back to 2. If yes: save and log.

Hygiene:
- No SELECT * — specify columns.
- LIMIT on exploratory queries.
- Always parameterized — never concatenate user input into SQL strings.
- Use cost guardrails if the database supports them.

## Error handling

On failure: show the error, explain in plain language, propose a fix, wait for approval.
Never silently retry. If retrying, vary your approach.
After two failed attempts, stop and ask.

## Anti-patterns

- Do not hard-code environment-specific values (IDs, paths, table names).
- Do not produce unrequested output.
- Do not fabricate file contents, API responses, or schema details you haven't verified.
- Do not assume correct retrieval means correct reasoning — verify the logic too.

## Context & memory

- Files in `./context/` auto-load every session and are READ-ONLY. Never modify originals.
- `/memory add` persists facts across sessions.
- `/restore` reverts file changes (checkpointing enabled).
- Quality degrades as context grows. Compact or start fresh for new task phases.
- For multi-phase work: write findings to files between phases so context can be reset.

## Log formats summary

Session log:  `[YYYY-MM-DD HH:MM] <what you did>` — narrative
Query log:    `[YYYY-MM-DD HH:MM] RAN: <name> | Rows: <count> | Result: <summary>`
Audit log:    `[YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>` — structured
