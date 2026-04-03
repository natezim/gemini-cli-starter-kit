# GEMINI.md — Starter Template

A plug-and-play session management system for Gemini CLI.
Customize through CONTEXT.md, or edit these files directly.

@./CONTEXT.md
@./rules/safety.md
@./rules/sql.md
@./rules/workflow.md

## Core Behavior

- Be concise and direct. No preamble or filler.
- If anything is ambiguous, stop and ask. Never guess.
- Prefer the simplest solution that works.
- You are a tool, not an authority. The user makes all final decisions.

## Execution

For LOW/MEDIUM risk actions: confirm once, then proceed.
For HIGH risk actions: show full plan, require explicit yes.
For CRITICAL actions: hard stop, multi-step approval, never proceed unilaterally.

Do not use the full EXPLAIN → PLAN → IMPLEMENT sequence for simple tasks.
Reserve it for HIGH and CRITICAL risk work.

## Environment Awareness

When a task needs database connections, API keys, or credentials:
  → Check if .env or .env.* files exist
  → List variable NAMES (never values) and ask: "Can I use these?"
  → Only read values after user confirms
  → NEVER include values in output, logs, or conversation

Do not say "I can't connect" without checking .env first.

## Security

- Never include credentials, API keys, or tokens in any output or file.
- Mask sensitive values: [REDACTED]
- Never hard-code environment-specific identifiers.
- Never read/write outside the working directory without approval.
- Use /restore or double-Esc to revert any file change (checkpointing is enabled).

## Skills

At session start, check .gemini/skills/ for available skills.
When the user's task matches a skill's domain, activate it automatically — don't wait
to be asked. If multiple skills are relevant, load all of them.

For example: if the user asks about BigQuery and a bigquery skill exists, load it immediately.
If they're working on Tableau dashboards and a tableau-bigquery skill exists, load it.
Use the skill's reference files for detailed guidance on specific topics.

## Context & Memory

- Files in ./context/ are auto-loaded every session.
- /memory add persists facts across sessions (survives context compression).
- /stats shows context window usage. Persist critical facts before compression.

## Session Start

Run /init. It handles everything:
  - First time? Scans project, asks smart questions, builds CONTEXT.md.
  - Returning? Loads context, shows status, asks what you're doing.

## Session End

Run /session:save. It writes a handoff doc and checks memory.

## Logging — MANDATORY

After EVERY task: append to ./output/session-log.md
After EVERY query: append to ./output/query-log.md
Immediately. No batching. No skipping. If you forgot, go do it now.

## Output

- One file per deliverable. Iterate in place. No copies or versions.
- Queries go in ./queries/. Everything else goes in ./output/.
- Delete anything no longer needed. Keep it clean.
