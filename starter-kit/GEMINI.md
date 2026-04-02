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

Read .env and .env.* files at session start to know what's available.
Use the variable NAMES to understand your capabilities — which databases, APIs,
and services are configured. Use the VALUES when connecting or running queries.
NEVER include values in output, logs, files, or conversation. Reference by variable name only.

Examples:
  DATABASE_URL is set     → you can connect to Postgres
  BIGQUERY_PROJECT is set → you can run BigQuery queries
  OPENAI_API_KEY is set   → you can call the OpenAI API

If a user asks you to do something and the credentials are in .env, just do it.
Don't say "I can't" — check .env first.

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

Applies to any database, query tool, API, CLI, or data system.
Project-specific tools and connections are defined in CONTEXT.md.

Before running any query or command:
  → Show it in full and wait for confirmation.
  → If a dry run or preview mode exists, use it first and report the result.
  → Report any cost, row count, or impact estimate before executing.
  → If estimated impact exceeds limits in CONTEXT.md, stop and ask.

Hard rules:
  → Only run read operations unless explicitly told otherwise.
  → Never modify, delete, or write to production systems without CRITICAL-level approval.
  → Never use wildcard selects on unknown data sources. Always specify fields.
  → Always apply row or result limits on exploratory queries.

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
