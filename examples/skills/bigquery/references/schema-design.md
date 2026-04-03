# Schema Design

## Native JSON type — columnar shredding

- Stored in "shredded" columnar format — each JSON key becomes a virtual column.
- **Only referenced paths are scanned and billed.**
- Significantly faster/cheaper than JSON-as-STRING (full scan + parse).
- Cannot partition or cluster on JSON columns.
- Number values auto-type as FLOAT64.
- Best for semi-structured data: avoids rigid schemas while retaining columnar performance.

## STRUCT/ARRAY denormalization

- Eliminates JOIN shuffling. Benchmarks show **~2x** query speed.
- STRUCTs support up to 15 nesting levels.
- Can cluster on STRUCT fields: `user.user_id`.
- Cannot cluster on fields inside ARRAYs.
- `CROSS JOIN UNNEST` drops empty arrays; `LEFT JOIN UNNEST` preserves them.

```sql
-- Denormalized schema example
CREATE TABLE orders (
  order_id INT64,
  customer STRUCT<id INT64, name STRING, segment STRING>,
  items ARRAY<STRUCT<product_id INT64, name STRING, qty INT64, price FLOAT64>>
)
```

## Search indexes — sub-second text search

```sql
CREATE SEARCH INDEX my_index ON dataset.logs (message)
OPTIONS (analyzer = 'LOG_ANALYZER')
```

Analyzers:
- **LOG_ANALYZER**: default, machine-generated data.
- **NO_OP_ANALYZER**: exact match only.
- **PATTERN_ANALYZER**: custom regex tokenization.

Optimizes: `SEARCH()`, `=`, `IN`, `LIKE`, `STARTS_WITH`.
Index storage: typically 50-100% of indexed column size. Maintenance free.
1 TB table: without index scans terabyte; with index reads megabytes.

## PK/FK constraints — join elimination

```sql
ALTER TABLE dataset.orders ADD PRIMARY KEY (order_id) NOT ENFORCED;
ALTER TABLE dataset.items ADD FOREIGN KEY (order_id)
  REFERENCES dataset.orders(order_id) NOT ENFORCED;
```

- NOT ENFORCED — BigQuery doesn't validate, but optimizer uses for:
  - Join elimination (removes unnecessary joins)
  - Outer join elimination
  - Join reordering
- Zero-cost optimization with measurable improvements on star schemas.

## Wildcard tables

- Matches up to 1,000 tables. Cannot match views.
- Results **never cached**.
- **Always filter `_TABLE_SUFFIX`** — without it, all matching tables scanned.
- Partitioned tables outperform sharded tables in planning and support DML + caching.

```sql
SELECT * FROM `project.dataset.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20240101' AND '20241231'
```

## Table clones and snapshots

- **Clones**: writable, copy-on-write. Initially zero storage cost.
- **Snapshots**: read-only backups, same cost model.
- Both support time-travel (up to 7 days) and expiration.
- Re-clustering base table causes clones to be charged full storage.

```sql
CREATE TABLE dev.orders CLONE prod.orders;
CREATE SNAPSHOT TABLE backups.orders_snap CLONE prod.orders;
CREATE OR REPLACE TABLE prod.orders CLONE backups.orders_snap;  -- restore
```
