# Data & Command Safety

Read-only by default. Classify every action before running it.

## Action Classification

SAFE — proceed without asking:
  Read operations (SELECT with LIMIT, SHOW, DESCRIBE, EXPLAIN), dry runs, previews

MEDIUM — confirm once:
  Creating new objects (tables, views) in non-production environments
  Writing to staging/test systems

CRITICAL — require explicit, specific approval:
  Any modification to production (ALTER, DROP, TRUNCATE, DELETE, UPDATE, MERGE)
  Any permission change (GRANT, REVOKE)
  Any DDL against production tables, views, or schemas

## Hard Rules

- Never modify production systems unless the user spells out exactly what to change and confirms.
- "Clean up the database" is not approval to DROP anything.
- Never use SELECT * — always specify columns.
- Always use LIMIT on exploratory queries.
- Validate or dry-run before running expensive operations when the tool supports it.
- Estimate impact before running any data-modifying command.
- If a database supports cost guardrails (byte limits, query governors), use them.

## Before Running Any Command

1. Show the full command/query.
2. Report estimated impact (rows affected, cost if applicable).
3. Wait for confirmation.
4. If impact exceeds limits in CONTEXT.md, stop and ask.

## Error Handling

On failure:
1. Show the full error message.
2. Explain the likely cause in plain language.
3. Propose a fix and wait for approval before retrying.
4. Never silently retry.
5. After two failed attempts, stop and ask for guidance.

## Bulk Operations

Any operation affecting multiple files or running in a loop:
list every affected item and get explicit confirmation first.

## Anti-Patterns

- Do not silently retry failed commands.
- Do not hard-code environment-specific values (IDs, paths, table names, endpoints).
- Do not take CRITICAL actions without explicit multi-step approval.
- Do not produce output the user did not request.
- Do not guess at missing context. Stop and ask.
