# SQL & Query Management

## Query Library

All queries live in ./queries/ as .sql files. One file per query, iterated in place.

Writing queries:
  → Save with a clear name (e.g., top-expensive-jobs.sql, daily-revenue.sql)
  → Include a comment header: purpose, target table/dataset, date last modified
  → Iterate the same file — never create copies or versions

## Testing — Be Self-Sufficient

After writing a query:
1. Run --dry_run or EXPLAIN to check bytes scanned and validate syntax.
2. Run with LIMIT to verify columns and results look right.
3. Fix any issues yourself before saving — don't hand back broken SQL.
4. Only save to ./queries/ once verified working.
5. Report: row count, bytes scanned, estimated cost, any issues found.

## Execution Logging

After EVERY query execution, IMMEDIATELY append to ./output/query-log.md:
  [YYYY-MM-DD HH:MM] RAN: <filename> | Rows: <count> | Bytes: <scanned> | Cost: ~$<est>

## Query Hygiene

- Never use SELECT * — always specify columns.
- Always use LIMIT on exploratory queries.
- Prefer BigQuery Views over Custom SQL (preserves join culling and query fusion).
- LIMIT does not reduce bytes scanned on standard (non-clustered) tables.
- Partition filters must use constant expressions to trigger pruning.
  ✓ WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  ✗ WHERE EXTRACT(YEAR FROM event_date) = 2024

## When User Asks "What Queries Do We Have"

List contents of ./queries/ with a one-line description from each file's header comment.

## Reproducibility

- Include the exact command/query used to produce any result.
- Output files should be self-contained enough to rerun without session context.
