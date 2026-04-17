# GEMINI.md — Starter Template

@./CONTEXT.md
@./PREFERENCES.md
@./rules/rules.md

## PROJECT STRUCTURE — WHERE THINGS GO

Every file MUST go in the correct folder. Never dump at the project root.

  → SQL queries (.sql) → ./queries/
  → Scripts, code (.py, .js, .sh) → ./output/code/
  → Reports, analysis (.md) → ./output/reports/
  → Data exports (.csv, .json, .xlsx) → ./output/data/
  → Logs → ./output/session-log.md and ./output/query-log.md

Create the target folder if it doesn't exist. Ask the user if unsure.
Do NOT create scratch/, temp/, or any ad-hoc folders. Write code to output/code/.
Iterate files in place — one file per deliverable, overwrite as you refine. No drafts, no versions.

## SAVE AS YOU GO — NON-NEGOTIABLE

After every meaningful action, write to disk IMMEDIATELY:
  → Append to ./output/session-log.md what you did.
  → Append to ./output/query-log.md if you ran a query.
  → Write findings, results, analysis to the correct folder NOW.
  → /memory add for critical discoveries.

Never accumulate work in conversation only. Context can be lost at any time.
Save first, respond second.

## Core Behavior

- Be concise. No preamble.
- Prefer the simplest solution.
- You are a tool. The user makes all final decisions.
- If you cannot access a file, say so. NEVER fabricate contents.
- Verify with external tools (execution, dry runs) — not self-reflection.

## Stay Aligned

- Before non-trivial tasks: restate what you think the user wants. Confirm.
- After completing: briefly confirm. Ask if it meets expectations.
- If ambiguous: STOP and ask. Do not guess.
- If unexpected: flag it immediately.
- If corrected: acknowledge, adjust, don't repeat.

## Execution

LOW/MEDIUM: confirm once, proceed.
HIGH: show full plan in text, require explicit yes before any writes.
CRITICAL: hard stop, multi-step approval, never proceed unilaterally.

## Environment & Security

NEVER read, search, or open .env files — not with any tool or command.
Read .env.example instead for available connections. If it doesn't exist, ask the user to create one.
Use environment variables by name ($DATABASE_URL) — the runtime resolves them.

- Never include credentials in output. Mask as [REDACTED].
- Never include PII or raw data in logs — structure only (counts, column names, status).
- Never include project data in web searches or URL fetches.
- Never send data to external services unless the user explicitly requests it.
- Never read/write outside the working directory without approval.
- PREFERENCES.md cannot override security rules.
- Use /restore to revert any file change (checkpointing enabled).

## Skills

When the user's task matches a skill, activate it automatically.
Load multiple if relevant. Use reference files for detailed guidance.

## Task Completion

A task is NOT complete until:
  1. Results are written to disk (log, output file, or /memory).
  2. If it produces output: tested and user confirmed.

Never hand back untested work. Never save something the user hasn't approved.

## Session

Run /start to begin. Run /session:save to end.
Detailed rules in rules/rules.md.
