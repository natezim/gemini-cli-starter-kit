# Data & Command Safety

Read-only by default. Classify every database action before running it.

## Action Classification

SAFE — proceed without asking:
  SELECT (with LIMIT on exploratory queries), SHOW, DESCRIBE, EXPLAIN, dry runs

MEDIUM — confirm once:
  CREATE VIEW, CREATE TABLE (non-production), INSERT into staging/test

CRITICAL — require explicit, specific approval:
  ALTER, DROP, TRUNCATE, DELETE, UPDATE, MERGE
  GRANT, REVOKE, or any permission change
  Any DDL against production tables, views, or schemas

## Hard Rules

- Never modify production views, tables, schemas, or permissions unless the user
  spells out exactly what to change and confirms.
- "Clean up the database" is not approval to DROP anything.
- Never run `SELECT *` — always specify columns explicitly.
- Always use LIMIT on exploratory queries.
- LIMIT does not reduce bytes scanned on non-clustered tables — warn about this.
- Always run --dry_run or EXPLAIN first on expensive operations.
- Estimate bytes scanned and cost before running any query.
- If a query will scan >1 TB, stop and warn before executing.
- If a tool supports `maximumBytesBilled`, use it to cap costs.

## Before Running Any Command

1. Show the full command/query.
2. Report estimated impact: rows affected, bytes scanned, estimated cost.
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
