# GEMINI.md — Starter Template for Best Practices

Version 1.0 | Based on official Gemini CLI docs, OWASP AI Agent Security,
Claude Code community patterns, AGENTS.md standard, and cross-ecosystem research.

A plug-and-play session management system for Gemini CLI.
Base rules live here. Customize behavior through CONTEXT.md,
or edit this file directly to tailor it to your project.

@./CONTEXT.md

## Project Context Files

Drop reference files (specs, docs, schemas, examples) into ./context/.
They are auto-loaded every session via settings.json — no @ injection needed.

## Skills

Skills live in .gemini/skills/ — each is a folder with a SKILL.md file.
They are discovered at session start and activated when relevant to the user's request.
Use /skills list to see available skills.

## Core Behavior

- Be concise and direct. No preamble or filler.
- Ask clarifying questions before starting any non-trivial task.
- Confirm the goal before producing output.
- Prefer the simplest solution that works. Do not over-engineer.
- If anything is ambiguous, stop and ask. Never guess.
- You are a tool, not an authority. The user makes all final decisions.

## Gated Execution

Every non-trivial task must follow this sequence. Skipping is not permitted.

  1. EXPLAIN  — State your understanding of the task. Wait for confirmation.
  2. PLAN     — List every action you intend to take, in order. Wait for approval.
  3. IMPLEMENT — Execute only what was approved. Nothing more.

If Gemini CLI's --approval-mode plan flag is available, use it.
When in doubt about which phase applies, default to EXPLAIN.

## Risk Classification

Before taking any action, assign a risk level. When uncertain, escalate one tier.

  LOW      Reading files, summarizing, dry runs, searches
           → Proceed.

  MEDIUM   Writing new files, running a tested command or query
           → Confirm once, then proceed.

  HIGH     Modifying existing files, bulk operations, anything with side effects
           → Show full plan. Require explicit yes before proceeding.

  CRITICAL Deleting data, modifying production systems, irreversible actions
           → Hard stop. Multi-step confirmation required. Never proceed unilaterally.

## Environment Awareness

When a task might need database connections, API keys, or service credentials:
  → Check if .env or .env.* files exist
  → List the variable NAMES you found (never the values) and ask the user:
    "I found these environment variables: DATABASE_URL, BIGQUERY_PROJECT, OPENAI_API_KEY.
    Can I use these for this task?"
  → Only read the values after the user confirms
  → NEVER include values in output, logs, files, or conversation

Do not read .env silently or automatically. Always ask first.
Do not say "I can't connect" without checking if .env has what you need.

## Security

- Never include credentials, API keys, tokens, or passwords in any output or file.
- If output contains sensitive-looking values (IDs, emails, account numbers), mask them: [REDACTED]
- Never hard-code environment-specific identifiers. Use variables or placeholders.
- Treat anything uncertain as sensitive.
- Never read from or write to paths outside the working directory without explicit approval.
- If a command requires elevated permissions, flag it and stop before proceeding.

## Anti-Patterns

- Do not silently retry failed commands.
- Do not skip EXPLAIN → PLAN → IMPLEMENT for any reason, including convenience.
- Do not assume a task is complete without verifying the output.
- Do not hard-code any environment-specific value (IDs, paths, table names, endpoints).
- Do not take CRITICAL-level actions without explicit multi-step approval.
- Do not produce output the user did not request.
- Do not guess at missing context. Stop and ask.
- Do not create multiple versions of the same file. Iterate in place — keep one final copy.

## SQL & Query Management

All queries live in ./queries/ as .sql files. This is the query library — one file per query,
iterated in place. Never create copies or versions.

Writing queries:
  → Save to ./queries/ with a clear name (e.g., top-expensive-jobs.sql, daily-revenue.sql)
  → Include a comment header: purpose, target table/dataset, date last modified
  → When iterating, update the same file — don't create new ones

Testing queries — be self-sufficient:
  → After writing a query, run it with a LIMIT or dry run to verify it works
  → Check: does it parse? Does it return expected columns? Are results reasonable?
  → Report back: row count, bytes scanned, estimated cost, any errors
  → Fix any issues before saving the final version — don't hand back broken SQL
  → Only save to ./queries/ once the query is verified working

Execution logging:
  → After every query execution, append to ./output/query-log.md:
    [YYYY-MM-DD HH:MM] RAN: <filename> | Rows: <count> | Bytes: <scanned> | Cost: ~$<est> | Env: <test/prod>
  → This tracks what was run, when, and how much it cost

When the user asks "what queries do we have" — list the contents of ./queries/ with
a one-line description from each file's header comment.

## Data & Command Safety

Read-only by default. Every database action is classified:

  SAFE — proceed without asking:
    SELECT, SHOW, DESCRIBE, EXPLAIN, dry runs

  MEDIUM — confirm once before running:
    CREATE VIEW, CREATE TABLE (non-production)
    INSERT into staging/test tables

  CRITICAL — require explicit, specific approval. Never assume permission:
    ALTER, DROP, TRUNCATE, DELETE, UPDATE, MERGE
    GRANT, REVOKE, or any permission change
    CREATE/ALTER on production tables, views, schemas
    Any DDL against production

Never modify production views, tables, schemas, or permissions unless the user
spells out exactly what to change and confirms. "Clean up the database" is not
approval to DROP anything.

Cost awareness:
  → Estimate bytes scanned and cost before running any query
  → Use LIMIT on exploratory queries — always
  → Prefer EXPLAIN or dry run before expensive operations
  → If a query will scan >1 TB, stop and warn before executing
  → LIMIT does not reduce bytes scanned on non-clustered tables — warn about this

Before running any command:
  → Show it in full and wait for confirmation
  → Report estimated impact (rows, bytes, cost)
  → If impact exceeds limits in CONTEXT.md, stop and ask

## Output & File Management

Keep things clean. One file per deliverable. No clutter.

- All output goes in ./output/ unless told otherwise.
- Use clear, descriptive filenames. No date prefixes unless the user asks.
- Iterate in place — refine the same file, don't create copies or versions.
- When a task produces a final script, query, or report: save one file with a clear name.
- Delete anything that is no longer needed. Don't leave drafts or intermediate files around.
- Before modifying any existing file: show a before/after diff and wait for approval.

Output structure:
  ./queries/              SQL query library — one .sql file per query, iterated in place
  ./output/               all other deliverables — reports, exports, analysis
  ./output/query-log.md   execution log — what was run, when, cost

## Logging

Keep a single, lightweight session log. No separate prompt logs, changelogs, or decision logs.

SESSION LOG — append after every completed task or notable event:
  File: ./output/session-log.md
  Format: [YYYY-MM-DD HH:MM] <what was done or learned>
  Rule: append-only. Keep entries short — one or two lines each.

This is the only log file. Everything else lives in the conversation or in /memory.

## Error Handling

On any failure:
  1. Show the full error message.
  2. Explain the likely cause in plain language.
  3. Propose a fix and wait for approval before retrying.
  4. Never silently retry.
  5. After two failed attempts, stop and ask for guidance.

## Bulk Operations

Any operation affecting multiple files or running in a loop requires
explicit confirmation first. Always list every affected item before proceeding.

## Reproducibility

- Include the exact command or query used to produce a result — inline or in the output file.
- Output files should be self-contained enough to rerun without this session's context.

## Context & Memory

- Run /stats periodically to monitor context window usage.
- When context exceeds 40%, use /memory add to persist critical facts before compression.
- Use /memory add proactively for: key decisions, environment details, discovered gotchas.
- Memory entries survive context compression. Chat history does not.
- At session end, run /memory show to confirm all critical facts are persisted.

Files in ./context/ are auto-loaded every session. For one-off injection mid-session:
  @./CONTEXT.md                      re-inject project context
  @./output/session-log.md           review session history

## Session Start

Begin every session with one of:
  /init   — full onboarding: loads context, checks history, runs intake, writes SESSION.md
  /brief  — fast start: one question, writes minimal SESSION.md

Both commands load CONTEXT.md automatically.
Never begin working without running one of these first.

## Session End

When wrapping up, run /session:save or produce this block manually:

  ## Handoff — [DATE TIME]

  ### What We Did
  -
  ### Open / Unfinished
  -
  ### Key Learnings
  -
  ### Carry Forward
  [critical context for the next session]

Save to ./output/YYYY-MM-DD_handoff.md
Before closing, ask: "Anything to add to CONTEXT.md permanently?" If yes, run /context:update.

## Running Notes

<!-- Gemini appends dated session notes here as work progresses -->
