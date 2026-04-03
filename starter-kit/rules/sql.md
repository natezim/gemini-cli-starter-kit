# SQL & Query Management

## Query Library

All queries live in ./queries/ as .sql files. One file per query, iterated in place.

Writing queries:
  → Save with a clear name (e.g., top-expensive-jobs.sql, daily-revenue.sql)
  → Include a comment header: purpose, target table/dataset, date last modified
  → Iterate the same file — never create copies or versions

## Query lifecycle — a query is NOT done until the user confirms it

Every query follows this cycle. Do not skip steps. Do not save until confirmed.

  1. WRITE the query.
  2. VALIDATE — check syntax. Use EXPLAIN or a dry run if the database supports it.
  3. TEST — run it and verify: correct columns, reasonable results, no errors.
     If something is wrong, fix it and go back to step 2.
  4. SHOW the user the results and the query. Ask: "Does this look right?"
  5. If the user says no or wants changes: adjust and go back to step 2.
  6. If the user confirms: save to ./queries/ and log the execution.

A query is not saved until the user says it's correct. Do not assume.
If the user doesn't explicitly confirm, ask.

## Execution Logging

Logging is part of query completion — not a separate step.
After EVERY query execution, append to ./output/query-log.md:
  [YYYY-MM-DD HH:MM] RAN: <filename or description> | Rows: <count> | Result: <summary>

## Query Hygiene

- Never use SELECT * — always specify columns.
- Always use LIMIT on exploratory queries.
- Estimate cost or impact before running expensive queries.
- If the database supports cost guardrails (e.g., byte limits), use them.

## When User Asks "What Queries Do We Have"

List contents of ./queries/ with a one-line description from each file's header comment.

## Reproducibility

- Include the exact command/query used to produce any result.
- Output files should be self-contained enough to rerun without session context.
