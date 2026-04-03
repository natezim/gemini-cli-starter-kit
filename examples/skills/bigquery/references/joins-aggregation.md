# Joins & Aggregation

## Join optimization

**Broadcast vs hash join**: threshold not publicly documented.
- Tables under ~1 GB very likely broadcast.
- Execution plan: `JOIN EACH WITH ALL` = broadcast, `JOIN EACH WITH EACH` = hash/shuffle.
- No join hint syntax exists. Strategy: pre-filter, project only needed columns, pre-aggregate.

**INT64 vs STRING keys**: INT64 is 8 bytes fixed; STRING is 2 + UTF-8 length.
INT64 is **1.2-1.4x faster**. For STRING keys, use `FARM_FINGERPRINT` for faster INT64 hashes.

**NOT EXISTS beats NOT IN**:
- NOT IN has NULL trap: if subquery contains NULL, returns zero rows silently.
- NOT EXISTS short-circuits, handles NULLs correctly. Always prefer it.

**PK/FK constraints enable join elimination**:
```sql
ALTER TABLE dataset.orders ADD PRIMARY KEY (order_id) NOT ENFORCED;
ALTER TABLE dataset.order_items ADD FOREIGN KEY (order_id) REFERENCES dataset.orders(order_id) NOT ENFORCED;
```
Zero-cost optimization. Enables join elimination, outer join elimination, join reordering.
Combined with History-Based Optimizations, produces optimal plans.

**No LATERAL JOIN** — use UNNEST:
- `CROSS JOIN UNNEST(array)` drops rows with empty arrays.
- `LEFT JOIN UNNEST(array)` preserves them.

## HLL++ sketches — 93% cost reduction for distinct counts

```sql
-- Build daily sketches
SELECT date, HLL_COUNT.INIT(user_id) AS sketch
FROM events GROUP BY date

-- Merge over rolling window
SELECT date, HLL_COUNT.EXTRACT(
  HLL_COUNT.MERGE(sketch) OVER (ORDER BY date ROWS 6 PRECEDING)
) AS weekly_uniques
FROM daily_sketches
```

- Default precision 15 = ~0.57% error at 95% confidence.
- Sketch for 50M+ records is ~32KB.
- Most powerful cost optimization for recurring distinct-count queries.

## GROUPING SETS — single scan replaces multiple GROUP BYs

```sql
SELECT region, product, SUM(revenue)
FROM sales
GROUP BY GROUPING SETS (
  (region, product),
  (region),
  ()
)
```

- Replaces multiple UNION ALL of GROUP BY queries with single scan.
- Use `GROUPING()` function to distinguish rollup NULLs from data NULLs.
- ROLLUP for hierarchical; CUBE for cross-dimensional.

## Window functions

**ROWS vs RANGE frames**:
- ROWS counts physical row offsets — fast, deterministic.
- RANGE compares ORDER BY values logically — handles ties, requires single numeric ORDER BY.
- Default with ORDER BY: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

**Window specification reuse**: each unique PARTITION BY/ORDER BY forces re-shuffle.
Use named WINDOW clause to share definitions:
```sql
SELECT
  SUM(x) OVER w,
  AVG(x) OVER w
FROM t
WINDOW w AS (PARTITION BY region ORDER BY date)
```

**QUALIFY** — filter on window results without subqueries:
```sql
SELECT * FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY date DESC) = 1
```
Window function doesn't need to appear in SELECT — only in QUALIFY.

## APPROX functions

- `APPROX_COUNT_DISTINCT`: <1% error, 5-10x faster than exact.
- `APPROX_QUANTILES(col, 100)`: returns array of percentile values.
- `ARRAY_AGG(col ORDER BY x LIMIT n)`: BigQuery optimizes to track only top-N.
- Max array size: 10,000 elements.
