# BigQuery SQL optimization: advanced patterns beyond the basics

**BigQuery's optimization surface is far deeper than partitioning and clustering basics.** This reference covers 10 areas of expert-level, non-obvious findings — from partition pruning edge cases that silently cause full-table scans, to HLL++ sketches that cut distinct-count costs by 93%, to pipe syntax that shipped GA in February 2025. Every finding includes concrete SQL, explains what breaks without it, and cites a source. This is designed to feed directly into a Gemini CLI skill file for BigQuery + Tableau workflows.

---

## 1. Advanced partitioning and clustering

### Integer range partitioning boundaries and NULL handling

**Finding**: Integer range partitions use `GENERATE_ARRAY(start, end, interval)` where `start` is **inclusive** and `end` is **exclusive**. Values at or above `end` land in `__UNPARTITIONED__` — they won't benefit from pruning. NULLs go to a separate `__NULL__` partition.

**Example**:
```sql
CREATE TABLE `project.dataset.sales_partitioned` (
  store_id INT64 NOT NULL,
  product_name STRING,
  sale_amount NUMERIC
)
PARTITION BY RANGE_BUCKET(store_id, GENERATE_ARRAY(0, 10000, 100));

-- Always add +1 to end parameter if your max value equals end
-- e.g., max store_id = 9999 → GENERATE_ARRAY(0, 10000, 100)
```

**Why It Matters**: If `end` equals your maximum value, that value falls into `__UNPARTITIONED__`. The **4,000 partition limit** means `(end - start) / interval` must stay under 4,000. Profile data distribution first with `APPROX_QUANTILES` to choose even bucket boundaries.

**Reference**: https://cloud.google.com/bigquery/docs/creating-partitioned-tables

---

**Finding**: You can partition on STRING columns using a synthetic integer key via `FARM_FINGERPRINT`.

**Example**:
```sql
CREATE TABLE `project.dataset.events_partitioned`
PARTITION BY RANGE_BUCKET(partition_id, GENERATE_ARRAY(0, 4000, 1))
AS
SELECT *, ABS(MOD(FARM_FINGERPRINT(country_code), 4000)) AS partition_id
FROM `project.dataset.events_raw`;
```

**Why It Matters**: Provides a workaround when your primary filter column isn't an integer or date, though you must include the same hash expression in WHERE clauses for pruning to work.

**Reference**: https://cloud.google.com/bigquery/docs/partitioned-tables

### Hourly partitioning gives you 166 days maximum

**Finding**: With hourly partitioning, the 4,000-partition limit means approximately **166 days** of data (4000 ÷ 24). Only TIMESTAMP and DATETIME columns support hourly granularity — DATE columns do not. All partition boundaries are UTC.

**Example**:
```sql
CREATE TABLE `project.dataset.clickstream` (
  click_id STRING,
  user_id STRING,
  click_timestamp TIMESTAMP
)
PARTITION BY TIMESTAMP_TRUNC(click_timestamp, HOUR)
OPTIONS (
  partition_expiration_days = 150,
  require_partition_filter = TRUE
);

-- For data spanning >6 months, use daily partitioning + clustering:
CREATE TABLE `project.dataset.events_long_term` (
  event_id STRING, event_ts TIMESTAMP
)
PARTITION BY TIMESTAMP_TRUNC(event_ts, DAY)
CLUSTER BY event_ts;  -- sub-day block pruning via clustering
```

**Why It Matters**: Exceeding 4,000 hourly partitions causes write rejections. Non-UTC timezone users scan extra partitions at day boundaries — a single Berlin-local day spans two UTC daily partitions.

**Reference**: https://cloud.google.com/bigquery/docs/partitioned-tables

### Clustering follows a left-prefix rule and ignores characters past 1,024

**Finding**: Clustering columns act like a composite index — filtering benefits apply only on a **left prefix**. Clustering on `(A, B, C)` means filtering on just `B` and `C` provides zero block pruning. STRING columns are clustered on only the **first 1,024 characters**. Tables under **1 GB** see negligible clustering benefit.

**Example**:
```sql
-- ✅ GOOD: Filters on left prefix
SELECT * FROM orders_clustered
WHERE customer_id = 12345 AND product_id = 'SKU-001';

-- ❌ BAD: Skips first clustering column
SELECT * FROM orders_clustered
WHERE product_id = 'SKU-001' AND order_id = 999;
-- Zero block pruning benefit

-- Audit clustering configuration programmatically:
SELECT table_name, column_name, clustering_ordinal_position
FROM `project.dataset.INFORMATION_SCHEMA.COLUMNS`
WHERE clustering_ordinal_position IS NOT NULL
ORDER BY table_name, clustering_ordinal_position;
```

**Why It Matters**: Wrong column order means queries skipping the first column get zero clustering benefit. Put the **most frequently filtered** column first. For long strings (URLs, JSON), extract shorter keys.

**Reference**: https://cloud.google.com/bigquery/docs/querying-clustered-tables

### Auto-reclustering is free but ALTER TABLE doesn't recluster existing data

**Finding**: Auto-reclustering works like an LSM tree — completely free, autonomous, no slot consumption. However, **ALTER TABLE to add clustering does NOT recluster existing data**. Only new writes benefit. To recluster existing data, you must rewrite the table with `CREATE OR REPLACE TABLE ... AS SELECT`.

**Example**:
```sql
-- This only affects NEW data:
ALTER TABLE `project.dataset.events`
SET OPTIONS (clustering_columns=['user_id']);

-- To recluster existing data, rewrite:
CREATE OR REPLACE TABLE `project.dataset.events`
PARTITION BY event_date
CLUSTER BY user_id
AS SELECT * FROM `project.dataset.events`;
```

**Why It Matters**: Teams often ALTER a table to add clustering and assume it's working. Existing data remains unclustered. Compare dry-run estimated bytes vs actual bytes processed to measure clustering effectiveness — if they're roughly equal, clustering isn't helping.

**Reference**: https://cloud.google.com/bigquery/docs/clustered-tables

### Partition pruning edge cases: what silently causes full-table scans

**Finding**: ANY function, CAST, arithmetic, or transformation applied to the partition column **kills pruning**. The partition column must appear isolated on one side of the comparison. Subqueries as predicates also prevent pruning. However, `IN` lists with **constant values** and `OR` conditions with constants DO support pruning.

**Example**:
```sql
-- ❌ BREAKS: Function on partition column
WHERE customer_id + 1 BETWEEN 30 AND 50
-- ❌ BREAKS: Type mismatch (DATE partition with TIMESTAMP literal)
WHERE event_date = TIMESTAMP '2025-11-10 00:00:00'
-- ❌ BREAKS: Subquery filter
WHERE event_date = (SELECT MAX(event_date) FROM dataset.events)
-- ❌ BREAKS: MERGE with WHEN NOT MATCHED BY SOURCE

-- ✅ WORKS: IN with constants
WHERE event_date IN ('2026-01-01', '2026-01-15', '2026-02-01')
-- ✅ WORKS: OR with constants
WHERE event_date = '2026-01-01' OR event_date = '2026-02-01'

-- ✅ FIX for subquery: Pre-compute into a variable
DECLARE max_date DATE;
SET max_date = (SELECT MAX(event_date) FROM dataset.events);
SELECT * FROM dataset.events WHERE event_date = max_date;
```

**Why It Matters**: This is the #1 cause of unexpectedly expensive queries. The scripting variable workaround for subquery predicates is essential for incremental/dbt patterns. Type mismatches between DATE partitions and TIMESTAMP filters are especially common when BI tools generate queries.

**Reference**: https://cloud.google.com/bigquery/docs/querying-partitioned-tables

---

## 2. Advanced JOIN patterns

### No documented broadcast threshold — Dremel decides adaptively

**Finding**: BigQuery's broadcast vs hash join threshold is **not publicly documented**. Dremel starts shuffling both sides, and if one side finishes quickly and is small enough, it **cancels the second shuffle** and switches to broadcast mid-execution. Practical guideline: tables under ~1 GB are very likely broadcast. You can identify the strategy in the execution plan: **`JOIN EACH WITH ALL`** = broadcast, **`JOIN EACH WITH EACH`** = hash/shuffle join.

**Example**:
```sql
-- No hint syntax exists. Instead, make the dimension small:
WITH small_dim AS (
  SELECT user_id, plan_name
  FROM `project.dataset.dim_users`
  WHERE region = 'us'
)
SELECT e.user_id, d.plan_name
FROM `project.dataset.fact_events` e
JOIN small_dim d USING (user_id)
WHERE e.event_date >= '2026-01-01';
```

**Why It Matters**: Unlike Spark, **BigQuery has no documented join hint syntax** for forcing broadcast. You must write queries that create conditions for the optimizer to choose broadcast — pre-filter aggressively, project only needed columns, pre-aggregate.

**Reference**: https://cloud.google.com/bigquery/docs/best-practices-performance-compute

### INT64 joins are ~1.4x faster than STRING joins

**Finding**: Google's official documentation recommends INT64 over STRING for join keys. INT64 is always **8 bytes fixed-width**; STRING is 2 bytes + UTF-8 length. Benchmarks show **1.2–1.4x** speedup with INT64 keys. For billion-row tables, this compounds significantly.

**Example**:
```sql
-- ✅ FAST: INT64 join key (8 bytes)
SELECT f.*, d.category_name
FROM fact_sales f JOIN dim_products d ON f.product_id = d.product_id;

-- If stuck with STRING keys, FARM_FINGERPRINT produces faster INT64 hashes:
ON FARM_FINGERPRINT(f.product_sku) = FARM_FINGERPRINT(d.product_sku)
```

**Why It Matters**: On billion-row tables, a 1.4x speed difference translates directly to slot-time and cost savings. Always prefer INT64 surrogate keys for join columns.

**Reference**: https://cloud.google.com/bigquery/docs/best-practices-performance-compute

### NOT EXISTS beats NOT IN for anti-joins — and NOT IN has a NULL trap

**Finding**: Google officially recommends **NOT EXISTS** over NOT IN. NOT IN with a subquery containing NULL values **silently returns zero rows** — a dangerous semantic trap. NOT EXISTS short-circuits on first match and handles NULLs correctly.

**Example**:
```sql
-- ✅ BEST: NOT EXISTS
SELECT * FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- ❌ DANGEROUS: NOT IN (if orders has NULL customer_id → returns 0 rows)
SELECT * FROM customers c
WHERE c.customer_id NOT IN (SELECT customer_id FROM orders);
```

**Why It Matters**: The NULL trap in NOT IN is one of the most dangerous SQL bugs because it produces no error — just silently wrong results. NOT EXISTS generates a more efficient anti-join execution plan with less data shuffling.

**Reference**: https://cloud.google.com/bigquery/docs/best-practices-performance-compute

### PK/FK constraints enable join elimination — even unenforced

**Finding**: Defining **PRIMARY KEY and FOREIGN KEY constraints** (NOT ENFORCED) gives the optimizer cardinality information that enables three optimizations: **join elimination** (removing unnecessary joins entirely), **outer join elimination**, and **join reordering**. This is a zero-cost optimization that can dramatically improve multi-table star-schema queries.

**Example**:
```sql
ALTER TABLE `project.dataset.store_sales`
  ADD PRIMARY KEY(ss_item_sk, ss_ticket_number) NOT ENFORCED,
  ADD FOREIGN KEY(ss_customer_sk) REFERENCES customer(c_customer_sk) NOT ENFORCED;

-- This join gets ELIMINATED entirely:
SELECT ss.* FROM store_sales ss
INNER JOIN customer c ON ss.ss_customer_sk = c.c_customer_sk;
-- With constraints, optimizer removes the join!
```

**Why It Matters**: Google's blog showed measurable reductions in TPC-DS benchmarks. Combined with History-Based Optimizations (see Topic 8), this gives the optimizer the best chance to produce optimal plans.

**Reference**: https://cloud.google.com/blog/products/data-analytics/join-optimizations-with-bigquery-primary-and-foreign-keys/

### BigQuery has no LATERAL JOIN — use UNNEST instead

**Finding**: BigQuery does **not** support the `LATERAL JOIN` keyword. The equivalent is correlated `CROSS JOIN UNNEST` or `LEFT JOIN UNNEST` on array columns. Critical difference: `CROSS JOIN UNNEST` drops rows with empty arrays; `LEFT JOIN UNNEST` preserves them.

**Example**:
```sql
-- Equivalent of LATERAL JOIN:
SELECT o.order_id, item.name, item.quantity
FROM `project.dataset.orders` o,
UNNEST(o.items) AS item;

-- Preserve rows with empty arrays:
SELECT o.order_id, item.name
FROM `project.dataset.orders` o
LEFT JOIN UNNEST(o.items) AS item;

-- Correlated ARRAY subquery:
SELECT order_id,
  ARRAY(
    SELECT AS STRUCT item.name, item.price * item.quantity AS line_total
    FROM UNNEST(items) AS item
    WHERE item.quantity > 0
  ) AS filtered_items
FROM orders;
```

**Why It Matters**: Understanding CROSS JOIN vs LEFT JOIN UNNEST prevents silently dropping rows with empty arrays — a common bug in GA4 BigQuery export analytics.

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

---

## 3. Window functions and QUALIFY patterns

### Tableau computes table calculations locally — push window functions via Custom SQL

**Finding**: Tableau's native connector pushes filters, aggregations, and JOINs to BigQuery. However, Tableau's **table calculations** (RANK, RUNNING_SUM, etc.) are computed locally after data retrieval, not pushed to BigQuery. To execute window functions on BigQuery's engine, use **Custom SQL** in your Tableau data source.

**Example**:
```sql
-- Custom SQL in Tableau to push window functions to BigQuery:
SELECT order_date, customer_id, revenue,
  RANK() OVER (PARTITION BY customer_id ORDER BY revenue DESC) AS revenue_rank
FROM `project.dataset.orders`
WHERE order_date >= '2025-01-01'
```

**Why It Matters**: Without Custom SQL, Tableau pulls all rows then ranks locally — expensive scan, slow computation, limited by Tableau memory. Custom SQL pushes computation to BigQuery's parallel engine.

**Reference**: https://help.tableau.com/current/pro/desktop/en-us/examples_googlebigquery.htm

### ROWS BETWEEN is faster; RANGE BETWEEN handles date gaps correctly

**Finding**: `ROWS` counts physical row offsets — fast and deterministic. `RANGE` compares ORDER BY key values logically — handles ties differently and requires a single numeric ORDER BY expression. **Default frame** with ORDER BY present is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which can produce surprising results via peer grouping.

**Example**:
```sql
-- ROWS: exact 7-row moving average (fast)
AVG(value) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)

-- RANGE: true 7-calendar-day average (handles missing dates)
AVG(value) OVER (ORDER BY UNIX_DATE(date) RANGE BETWEEN 6 PRECEDING AND CURRENT ROW)
```

**Why It Matters**: If dates have gaps, ROWS averages the last 7 available data points (could span weeks), while RANGE gives a true 7-day window. Use ROWS for speed; RANGE for calendar-correct time series.

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/window-function-calls

### Each unique window specification creates a separate shuffle stage

**Finding**: Multiple different PARTITION BY or ORDER BY clauses in window functions force BigQuery to re-shuffle data for each unique specification. Using the named `WINDOW` clause to share definitions reduces shuffle stages.

**Example**:
```sql
-- ✅ GOOD: Shared window definition — single shuffle
SELECT order_date, daily_revenue,
  SUM(daily_revenue) OVER w AS running_total,
  AVG(daily_revenue) OVER w AS running_avg,
  RANK() OVER w AS revenue_rank
FROM daily_revenue_table
WINDOW w AS (ORDER BY order_date);

-- ❌ BAD: Different partitions — multiple shuffle stages
SELECT *,
  SUM(x) OVER (PARTITION BY region ORDER BY date),
  AVG(y) OVER (PARTITION BY product ORDER BY date)
FROM t;
```

**Why It Matters**: Each shuffle stage consumes slots and time. High-cardinality PARTITION BY keys distribute work well; low-cardinality keys (e.g., 200 countries) create data skew. No PARTITION BY at all forces all data onto a single slot.

**Reference**: https://cloud.google.com/bigquery/docs/query-insights

### QUALIFY is syntactic sugar but dramatically improves readability

**Finding**: QUALIFY filters on window function results without subqueries. Evaluated after WINDOW in execution order. The window function doesn't need to appear in SELECT — it can be computed only in QUALIFY. Execution plan is identical to a subquery-based approach.

**Example**:
```sql
-- Top 3 products per category without exposing rank in output:
SELECT product_id, category, sales
FROM products
QUALIFY RANK() OVER (PARTITION BY category ORDER BY sales DESC) <= 3;

-- Second-to-last event per order:
SELECT order_id, event_ts, order_status
FROM order_events
WHERE order_status = 'order_updated'
QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY event_ts DESC) = 2;
```

**Why It Matters**: Eliminates CTEs/subqueries for window-based filtering. WHERE filters first, then QUALIFY — combine them for efficient two-stage filtering.

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

---

## 4. CTE materialization, recursive CTEs, and the UNION limit

### Non-recursive CTEs are NOT materialized — use temp tables instead

**Finding**: BigQuery does **not** materialize non-recursive CTEs. They are inlined and potentially **re-executed each time referenced**. There is no `MATERIALIZED` hint (unlike PostgreSQL). Google's official docs state: *"WITH clauses with CTEs are used for query readability, not performance."* The only reliable materialization mechanism is temporary tables (free storage, 24-hour lifetime).

**Example**:
```sql
-- ❌ CTE referenced twice = scanned twice
WITH expensive AS (
  SELECT user_id, COUNT(*) AS cnt
  FROM `project.dataset.large_table` GROUP BY user_id
)
SELECT a.user_id, a.cnt, b.cnt
FROM expensive a JOIN expensive b ON a.user_id = b.user_id;

-- ✅ Temp table forces materialization (free storage)
CREATE TEMP TABLE expensive AS
SELECT user_id, COUNT(*) AS cnt
FROM `project.dataset.large_table` GROUP BY user_id;
```

**Why It Matters**: Multi-referenced CTEs silently double costs. This is the single most impactful optimization choice in complex BigQuery queries.

**Reference**: https://cloud.google.com/bigquery/docs/best-practices-performance-compute

### Recursive CTEs exist, are materialized, and have a 500-iteration limit

**Finding**: `WITH RECURSIVE` is fully supported. Max recursion depth is **500 iterations** (default, increasable via Support). Recursive CTEs **are materialized** — the only CTEs that are. Each iteration runs as a separate execution stage.

**Example**:
```sql
WITH RECURSIVE org_chart AS (
  SELECT employee_id, name, manager_id, 1 AS level,
         CAST(name AS STRING) AS path
  FROM `project.hr.employees`
  WHERE manager_id IS NULL
  UNION ALL
  SELECT e.employee_id, e.name, e.manager_id,
         oc.level + 1, CONCAT(oc.path, ' > ', e.name)
  FROM `project.hr.employees` e
  INNER JOIN org_chart oc ON e.manager_id = oc.employee_id
  WHERE oc.level < 50  -- explicit depth guard
)
SELECT * FROM org_chart;
```

**Why It Matters**: Essential for hierarchical data (org charts, bill of materials, category trees). Always include a depth guard and cycle prevention. Each iteration consumes slots — expensive for deep recursion.

**Reference**: https://cloud.google.com/bigquery/docs/recursive-ctes

### The ~89 UNION limit and the 50M complexity quota

**Finding**: BigQuery has an internal **query complexity score quota of 50 million** per project. Approximately **89 UNION ALL** operations exhaust this limit. The error message: *"Resources exceeded during query execution: Not enough resources for query planning — too many subqueries or query is too complex."* This limit applies at planning time — even zero-byte queries can fail.

**Example**:
```sql
-- WORKAROUND: Batch into temp tables
CREATE TEMP TABLE batch1 AS
SELECT * FROM table1 UNION ALL ... UNION ALL SELECT * FROM table45;

CREATE TEMP TABLE batch2 AS
SELECT * FROM table46 UNION ALL ... UNION ALL SELECT * FROM table90;

SELECT * FROM batch1 UNION ALL SELECT * FROM batch2;

-- Or use wildcard tables instead of explicit UNIONs:
SELECT * FROM `project.dataset.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20260101' AND '20260331';

-- Monitor proximity to the limit:
SELECT query_info.resource_warning
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE resource_warning IS NOT NULL;
```

**Why It Matters**: Affects ETL pipelines combining many source tables and BI tools that auto-generate UNION queries (dbt, Dataform). Wildcard tables and temp-table batching are the standard workarounds.

**Reference**: https://cloud.google.com/bigquery/docs/troubleshoot-queries

### Correlated subqueries must be decorrelatable or they fail

**Finding**: BigQuery auto-decorrelates simple correlated subqueries into JOINs. When it can't, it throws: *"Correlated subqueries that reference other tables are not supported unless they can be de-correlated."* Subqueries with ORDER BY, LIMIT, complex multi-table correlations, or OUTER JOINs inside ARRAY subqueries typically cannot be decorrelated.

**Example**:
```sql
-- ✅ Works (simple scalar, decorrelatable):
SELECT username,
  (SELECT mascot FROM Mascots WHERE Players.team = Mascots.team) AS mascot
FROM Players;

-- ❌ Fails (OUTER JOIN in ARRAY subquery):
SELECT ARRAY(
  SELECT AS STRUCT b.Column
  FROM UNNEST(a.Parts) p
  LEFT OUTER JOIN TableB b ON b.JoinColumn = p.JoinColumn
) FROM TableA a;

-- ✅ Fix: Pre-join in CTE, then aggregate
WITH joined AS (
  SELECT a.id, b.Column
  FROM TableA a, UNNEST(a.Parts) p
  LEFT JOIN TableB b ON b.JoinColumn = p.JoinColumn
)
SELECT id, ARRAY_AGG(Column) FROM joined GROUP BY id;
```

**Why It Matters**: Always prefer JOINs or window functions over correlated subqueries. JOINs benchmarked at **~14 seconds** vs correlated scalar subqueries at **~45 seconds** on identical data.

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/subqueries

---

## 5. HLL++ sketches and advanced aggregation

### HLL_COUNT enables incremental distinct counts with 93% cost reduction

**Finding**: BigQuery's four HLL++ functions (`HLL_COUNT.INIT`, `MERGE`, `MERGE_PARTIAL`, `EXTRACT`) enable storable, mergeable sketches for approximate distinct counts. Default precision 15 gives **~0.57% error** at 95% confidence. A sketch for 50M+ records is only ~32KB. Real-world case: DoiT reduced a COUNT(DISTINCT) from 6.5TB scan to **16.25GB** using HLL sketches.

**Example**:
```sql
-- Build daily sketches
CREATE TABLE daily_sketches AS
SELECT date, HLL_COUNT.INIT(user_id, 15) AS sketch
FROM events GROUP BY date;

-- 30-day rolling distinct users from sketches
SELECT date,
  HLL_COUNT.EXTRACT(
    HLL_COUNT.MERGE_PARTIAL(sketch) OVER (
      ORDER BY date RANGE BETWEEN 29 PRECEDING AND CURRENT ROW
    )
  ) AS rolling_30d_users
FROM daily_sketches;
```

**Why It Matters**: Rolling-window distinct counts normally require rescanning raw data for each window. HLL sketches reduce this to tiny sketch merges — the most powerful cost optimization for recurring distinct-count queries.

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/hll_functions

### GROUPING SETS scan data once instead of multiple GROUP BYs

**Finding**: `GROUPING SETS`, `ROLLUP`, and `CUBE` are GA in BigQuery. They replace multiple `UNION ALL` of GROUP BY queries with a single scan. The New York Times reported a query reduction from **2+ hours to 10 minutes** with 96% slot consumption reduction.

**Example**:
```sql
SELECT region, product, month, SUM(revenue),
  GROUPING(region) AS is_region_agg,
  GROUPING(product) AS is_product_agg
FROM sales
GROUP BY GROUPING SETS ((region, product, month), (region, month), (region), ())
ORDER BY region, product, month;
```

**Why It Matters**: Replaces N separate queries with a single scan. Use `GROUPING()` to distinguish rollup NULLs from data NULLs. ROLLUP for hierarchical data; CUBE for cross-dimensional analysis.

**Reference**: https://cloud.google.com/blog/products/data-analytics/bigquery-advanced-aggregation-functions/

### APPROX_COUNT_DISTINCT has <1% error; ARRAY_AGG supports inline ORDER BY + LIMIT

**Finding**: `APPROX_COUNT_DISTINCT` uses HLL++ internally with **<1% documented error** (~0.5% typical), running **5–10x faster** than exact COUNT(DISTINCT). `ARRAY_AGG` supports ORDER BY and LIMIT within the function call — BigQuery optimizes this to track only top-N elements, not sort the entire group. Max array size: **10,000 elements**.

**Example**:
```sql
-- Fast dashboard with approximate functions
SELECT DATE_TRUNC(event_timestamp, HOUR) AS hour,
  APPROX_COUNT_DISTINCT(user_id) AS unique_users,
  APPROX_QUANTILES(page_load_ms, 100)[OFFSET(50)] AS median_load,
  APPROX_QUANTILES(page_load_ms, 100)[OFFSET(95)] AS p95_load
FROM page_views GROUP BY hour;

-- Top 5 sales per product in one aggregation pass
SELECT product_id,
  ARRAY_AGG(STRUCT(sale_date, sale_amount)
    ORDER BY sale_amount DESC LIMIT 5
  ) AS top_5_sales
FROM sales_data GROUP BY product_id;
```

**Why It Matters**: For dashboards on billion-row tables, approximate functions provide 99%+ accuracy with massive cost reduction. ARRAY_AGG with ORDER BY + LIMIT replaces window-function + QUALIFY patterns for top-N collection.

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/approximate_aggregate_functions

---

## 6. Reading BigQuery execution plans

### Full query plans are only available post-execution via API or Console

**Finding**: BigQuery has **no EXPLAIN statement**. The full per-stage execution plan is available through: (1) the Console **Execution Graph** tab, (2) `bq show -j --format=prettyjson <job_id>`, or (3) the `jobs.get()` REST API at `statistics.query.queryPlan`. `INFORMATION_SCHEMA.JOBS` provides only summary metrics (total_slot_ms, total_bytes_processed), not per-stage detail.

**Example**:
```bash
# Full plan via CLI
bq show -j --format=prettyjson <job_id>
```

**Why It Matters**: Unlike PostgreSQL, you must actually run the query to see the plan. Dry run only estimates bytes processed, not execution strategy. Use INFORMATION_SCHEMA.JOBS for fleet-wide monitoring; use the API/Console for per-query diagnosis.

**Reference**: https://cloud.google.com/bigquery/docs/query-plan-explanation

### Detecting skew, join strategy, and partition pruning from the plan

**Finding**: **Skew**: Compare `computeRatioMax` vs `computeRatioAvg` within a stage — if max is **>2–3x** average, one worker is bottlenecked. Non-zero `shuffleOutputBytesSpilled` confirms memory pressure. **Join strategy**: Step substeps show `JOIN EACH WITH ALL` (broadcast) or `JOIN EACH WITH EACH` (hash/shuffle). **Partition pruning**: Check `statistics.query.totalPartitionsProcessed` in the API response; compare total_bytes_processed against expected partition size.

**Example**:
```sql
-- Find slot-heavy queries and estimate average parallelism
SELECT job_id, query, total_slot_ms,
  TIMESTAMP_DIFF(end_time, start_time, MILLISECOND) AS duration_ms,
  SAFE_DIVIDE(total_slot_ms,
    TIMESTAMP_DIFF(end_time, start_time, MILLISECOND)) AS avg_slots
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
  AND job_type = 'QUERY'
ORDER BY total_slot_ms DESC LIMIT 10;

-- Detect slot contention for a specific job
SELECT
  ROUND(COUNTIF(period_estimated_runnable_units > 0) / COUNT(*) * 100, 1)
    AS pct_time_waiting_for_slots
FROM `region-us`.INFORMATION_SCHEMA.JOBS_TIMELINE
WHERE job_id = 'my_job_id';
```

**Why It Matters**: `total_slot_ms` is the core metric for capacity-based pricing. The timeline view reveals second-by-second consumption patterns and queued-work contention. Performance insights in the Console flag skew, slot contention, and high-cardinality joins automatically.

**Reference**: https://cloud.google.com/bigquery/docs/query-plan-explanation, https://cloud.google.com/bigquery/docs/information-schema-jobs-timeline

---

## 7. Scripting and procedural SQL reference

### DECLARE, SET, and variable scoping

**Finding**: Variables support all BigQuery types including ARRAY and STRUCT. Type can be inferred from DEFAULT. SET can populate multiple variables from a single subquery using `SELECT AS STRUCT`. Max variable size: **1 MB**; max total: **10 MB**. Variables are scoped to their enclosing BEGIN…END block.

**Example**:
```sql
DECLARE tags ARRAY<STRING> DEFAULT ['etl', 'prod', 'v2'];
DECLARE config STRUCT<region STRING, max_rows INT64>
  DEFAULT STRUCT('us-central1', 10000);

-- SET multiple variables from one subquery
DECLARE corpus_count INT64;
DECLARE word_count INT64;
SET (corpus_count, word_count) = (
  SELECT AS STRUCT COUNT(DISTINCT corpus), SUM(word_count)
  FROM `bigquery-public-data.samples.shakespeare`
  WHERE LOWER(word) = 'methinks'
);
```

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/scripting

### Control flow: IF, WHILE, LOOP, FOR…IN, labels

**Finding**: Max nesting: **50 levels**. BREAK and LEAVE are synonymous; CONTINUE and ITERATE are synonymous. Labels allow breaking out of outer loops from inner loops. `FOR…IN` iterates over query results — the loop variable is a STRUCT.

**Example**:
```sql
-- FOR...IN to run dynamic SQL per table
FOR tbl IN (
  SELECT table_name
  FROM `my_project.my_dataset.INFORMATION_SCHEMA.TABLES`
  WHERE table_type = 'BASE TABLE'
)
DO
  EXECUTE IMMEDIATE FORMAT(
    "SELECT '%s' AS tbl, COUNT(*) AS rows FROM `my_project.my_dataset.%s`",
    tbl.table_name, tbl.table_name
  );
END FOR;

-- Labels for outer-loop break
outer_loop: LOOP
  SET a = a + 1;
  inner_loop: WHILE b < 3 DO
    SET b = b + 1;
    IF a = 3 AND b = 2 THEN BREAK outer_loop; END IF;
  END WHILE inner_loop;
END LOOP outer_loop;
```

**Why It Matters**: Loops are serial — avoid row-by-row iteration on large datasets. FOR…IN is ideal for metadata-driven operations (INFORMATION_SCHEMA iteration).

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/scripting

### EXECUTE IMMEDIATE: dynamic SQL with USING parameters

**Finding**: Executes a single dynamic SQL string. USING passes **values** safely (like query parameters). For **identifiers** (table/column names), use `FORMAT()` — USING cannot parameterize identifiers. Cannot nest IF, LOOP, WHILE, BEGIN/END, or CALL inside EXECUTE IMMEDIATE.

**Example**:
```sql
-- Safe value binding with USING
DECLARE result INT64;
EXECUTE IMMEDIATE "SELECT @a * (@b + 2)" INTO result USING 1 AS a, 3 AS b;

-- Dynamic table name via FORMAT (validate/allowlist inputs!)
DECLARE table_suffix STRING DEFAULT '2024';
EXECUTE IMMEDIATE
  FORMAT("SELECT COUNT(*) FROM `project.dataset.events_%s`", table_suffix);
```

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/scripting

### Exception handling and multi-statement transactions

**Finding**: `BEGIN…EXCEPTION WHEN ERROR THEN…END` catches runtime errors. Available: `@@error.message`, `@@error.statement_text`, `@@error.formatted_stack_trace`, `@@error.stack_trace`. RAISE re-throws; `RAISE USING MESSAGE` throws custom errors. Transactions support INSERT/UPDATE/DELETE/MERGE/TRUNCATE. **Permanent DDL is not allowed** inside transactions. Limits: **100 tables**, **100,000 partition modifications** per transaction.

**Example**:
```sql
BEGIN
  BEGIN TRANSACTION;
  INSERT INTO orders (order_id, amount) VALUES ('ord-001', 150.00);
  UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 'prod-99';
  COMMIT TRANSACTION;
EXCEPTION WHEN ERROR THEN
  SELECT @@error.message AS txn_error;
  ROLLBACK TRANSACTION;
END;
```

**Why It Matters**: Unhandled errors auto-rollback transactions. Variables declared in BEGIN are NOT accessible in the EXCEPTION handler. Always include explicit ROLLBACK in exception handlers.

**Reference**: https://cloud.google.com/bigquery/docs/transactions

### System variables vs script variables

**Finding**: System variables (`@@prefixed`) are built-in. Key writable ones: `@@dataset_id`, `@@time_zone`, `@@query_label`. Key read-only ones: `@@project_id`, `@@script.bytes_processed`, `@@script.slot_ms`, `@@row_count`. In session mode, top-level DECLARE'd variables and temp tables persist across query submissions. Use `_SESSION.table_name` to explicitly reference temp tables and avoid collisions with permanent tables.

**Example**:
```sql
SET @@dataset_id = 'my_dataset';
SET @@time_zone = 'America/New_York';
SELECT @@script.bytes_processed AS bytes_so_far;

-- Session temp table reference
SELECT * FROM _SESSION.my_temp_table;
```

**Reference**: https://cloud.google.com/bigquery/docs/reference/system-variables

---

## 8. Major 2024–2025 BigQuery features

### Pipe syntax is GA and changes how you write SQL

**Finding**: Pipe syntax (`|>`) shipped **GA on February 4, 2025**. Queries can start with `FROM` and chain operators linearly. New pipe-specific operators: `EXTEND` (add columns), `AGGREGATE ... GROUP BY`, `SET`, `DROP`, `RENAME`. Same optimizer, same cost — purely syntactic.

**Example**:
```sql
FROM `bigquery-public-data.thelook_ecommerce.order_items`
|> WHERE status = 'Completed'
|> EXTEND DATE(created_at) AS order_date
|> AGGREGATE COUNT(*) AS total_orders, SUM(sale_price) AS total_sales
   GROUP BY order_date
|> ORDER BY total_sales DESC
|> LIMIT 5;
```

**Why It Matters**: Eliminates deep nesting and repetitive CTEs. Especially valuable for log analytics and iterative exploration. Legacy SQL is being deprecated June 1, 2026.

**Reference**: https://cloud.google.com/bigquery/docs/reference/standard-sql/pipe-syntax

### VECTOR_SEARCH and vector indexes are GA

**Finding**: `VECTOR_SEARCH` is a table-valued function for similarity search over `ARRAY<FLOAT64>` embedding columns. Supports COSINE, EUCLIDEAN, DOT_PRODUCT distance metrics. Two index types: **IVF** (best for small query batches) and **TreeAH** (best for large batches, uses Google's ScaNN technology). Indexes auto-refresh as base data changes.

**Example**:
```sql
CREATE OR REPLACE VECTOR INDEX my_index
ON `project.dataset.embeddings`(embedding)
OPTIONS (index_type = 'IVF', distance_type = 'COSINE',
         ivf_options = '{"num_lists": 500}');

SELECT query.title, base.title, distance
FROM VECTOR_SEARCH(
  TABLE `project.dataset.documents`, 'embedding',
  TABLE `project.dataset.query_docs`,
  top_k => 5, distance_type => 'COSINE'
);
```

**Why It Matters**: Enables RAG, semantic search, and entity resolution natively in BigQuery — no external vector database needed. Combined with `ML.GENERATE_TEXT` and `AI.GENERATE_EMBEDDING`, enables end-to-end GenAI pipelines in SQL.

**Reference**: https://cloud.google.com/bigquery/docs/vector-search

### Continuous queries, History-Based Optimizations, and non-incremental materialized views

**Finding**: **Continuous queries** (Preview for most targets; Spanner export GA) process streaming data via `APPENDS()` — stateless only (no JOINs/aggregations), minimum 100 slots. **History-Based Optimizations** (GA October 2024) learn from prior executions to auto-improve join ordering, pushdown, and semijoin reduction — up to **100x** observed improvement. **Non-incremental materialized views** (GA April 2024) support complex JOINs and subqueries via `allow_non_incremental_definition = true` with `max_staleness` control.

**Example**:
```sql
-- Non-incremental MV with complex JOIN
CREATE MATERIALIZED VIEW mv_report
OPTIONS (
  allow_non_incremental_definition = true,
  max_staleness = INTERVAL 4 HOUR,
  enable_refresh = true,
  refresh_interval_minutes = 60
) AS
SELECT a.customer_id, b.product_name, SUM(a.amount) AS total
FROM orders a JOIN products b ON a.product_id = b.id
GROUP BY 1, 2;
```

**Reference**: https://cloud.google.com/bigquery/docs/continuous-queries-introduction, https://cloud.google.com/blog/products/data-analytics/new-bigquery-history-based-optimizations-speed-query-performance

### Table clones and snapshots for zero-cost dev environments

**Finding**: **Clones** are writable, lightweight copies using copy-on-write — initially zero storage cost. **Snapshots** are read-only backups with identical cost model. Both support time-travel (up to 7 days back) and expiration timestamps.

**Example**:
```sql
-- Writable clone for dev/test
CREATE TABLE dev.orders_clone
CLONE prod.orders
OPTIONS (expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY));

-- Read-only snapshot for backup
CREATE SNAPSHOT TABLE backups.orders_snap
CLONE prod.orders
FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR);

-- Restore from snapshot
CREATE OR REPLACE TABLE prod.orders CLONE backups.orders_snap;
```

**Why It Matters**: Eliminates `CREATE TABLE AS SELECT *` anti-pattern. A 10 TB clone costs $0/month initially vs ~$200/month for a full copy. Critical warning: re-clustering the base table can cause clones to be charged full storage.

**Reference**: https://cloud.google.com/bigquery/docs/table-clones-intro

---

## 9. Cost optimization patterns

### Finding zombie tables with INFORMATION_SCHEMA

**Finding**: Cross-reference `TABLE_STORAGE` (sizes) with `JOBS_BY_PROJECT` (referenced_tables in query history — 180-day retention) to identify tables never queried.

**Example**:
```sql
WITH queried_tables AS (
  SELECT DISTINCT ref.dataset_id, ref.table_id
  FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    UNNEST(referenced_tables) AS ref
  WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND job_type = 'QUERY' AND state = 'DONE'
)
SELECT ts.table_schema, ts.table_name,
  ROUND(ts.total_logical_bytes / POW(1024, 3), 2) AS size_gb
FROM `region-us`.INFORMATION_SCHEMA.TABLE_STORAGE ts
LEFT JOIN queried_tables qt
  ON ts.table_schema = qt.dataset_id AND ts.table_name = qt.table_id
WHERE qt.table_id IS NULL AND ts.total_logical_bytes > 0
ORDER BY ts.total_logical_bytes DESC;
```

**Why It Matters**: Teams save 10–30% on storage by auditing and removing unused tables.

**Reference**: https://cloud.google.com/bigquery/docs/information-schema-tables

### Long-term storage timer resets on writes, not reads

**Finding**: The 90-day timer for long-term pricing **resets** on: DML writes, streaming inserts, loads, and copy operations. It does **NOT reset** on: queries/reads, exports, metadata updates, or creating views. For partitioned tables, each partition is tracked independently.

**Why It Matters**: A single UPDATE on a historical table resets it to active pricing. A 12 TiB table moving from long-term to active costs an extra ~$120K/year. Design pipelines to be append-only.

**Reference**: https://cloud.google.com/bigquery/pricing

### Logical vs physical storage billing

**Finding**: Physical billing charges on compressed data at 2x the logical unit rate, but compression ratios of **2:1 to 16:1** are common, making it cheaper in practice. Critical difference: physical billing charges time-travel and fail-safe bytes **separately**. Switch at the dataset level; must wait **14 days** between switches.

**Example**:
```sql
-- Compare costs before switching
SELECT table_schema,
  ROUND(SUM(active_logical_bytes)/POW(1024,3) * 0.02
    + SUM(long_term_logical_bytes)/POW(1024,3) * 0.01, 2) AS logical_cost,
  ROUND(SUM(active_physical_bytes)/POW(1024,3) * 0.04
    + SUM(long_term_physical_bytes)/POW(1024,3) * 0.02, 2) AS physical_cost,
  ROUND(SAFE_DIVIDE(SUM(total_logical_bytes),
    SUM(active_physical_bytes + long_term_physical_bytes)), 1) AS compression_ratio
FROM `region-us`.INFORMATION_SCHEMA.TABLE_STORAGE_BY_PROJECT
GROUP BY table_schema ORDER BY logical_cost DESC;

ALTER SCHEMA myproject.my_dataset SET OPTIONS (storage_billing_model = 'PHYSICAL');
```

**Why It Matters**: 20–70% savings are common for structured columnar data with low churn. Reduce time-travel window to 2 days under physical billing if data recovery isn't critical.

**Reference**: https://cloud.google.com/bigquery/docs/storage_overview

### Editions pricing: autoscaling in 50-slot increments

**Finding**: On-demand pricing is now **$6.25/TiB** (raised from $5.00). Editions slot pricing: Standard $0.04/slot-hr, Enterprise $0.06, Enterprise Plus $0.10. Autoscales in **50-slot increments** with 1-minute minimum billing and 60-second cooldown. Set baseline to 0 for dev/test to pay only for actual usage.

**Reference**: https://cloud.google.com/bigquery/pricing

---

## 10. Schema design for performance

### Native JSON type uses columnar shredding — only scans referenced paths

**Finding**: BigQuery's native `JSON` type stores data in a "shredded" columnar format where each JSON key becomes a virtual column. Only referenced paths are scanned and billed. Compared to JSON-as-STRING (full column scan + parse), native JSON is significantly faster and cheaper. Limitations: **cannot** partition or cluster on JSON columns; number values auto-type as FLOAT64.

**Example**:
```sql
CREATE TABLE events (
  event_id STRING, event_time TIMESTAMP,
  payload JSON
) PARTITION BY DATE(event_time);

-- Only scans the user_id and action paths:
SELECT event_id, payload.user_id, payload.action
FROM events
WHERE INT64(payload.user_id) = 123;
```

**Why It Matters**: For semi-structured data, native JSON avoids rigid schemas while retaining columnar performance. Use STRUCT for known stable schemas (faster); JSON for evolving schemas.

**Reference**: https://cloud.google.com/bigquery/docs/json-data

### STRUCT/ARRAY nesting eliminates JOINs and reduces shuffle by 30%+

**Finding**: Denormalizing into nested/repeated fields eliminates JOIN shuffling. Benchmarks show **~2x** query speed improvement on nested vs flat tables. STRUCTs support up to **15 nesting levels**. You can cluster on STRUCT fields (`user.user_id`) but NOT on fields inside ARRAYs.

**Example**:
```sql
-- Nested schema eliminates orders→items JOIN
SELECT o.order_id, item.product_name, item.quantity * item.unit_price AS line_total
FROM orders o, UNNEST(o.items) AS item
WHERE o.order_date >= '2026-01-01' AND item.quantity > 5;

-- STRUCT access needs no UNNEST:
SELECT order_id, shipping.address.city FROM orders;
```

**Reference**: https://cloud.google.com/bigquery/docs/best-practices-performance-nested

### Search indexes enable sub-second text search on terabyte tables

**Finding**: `CREATE SEARCH INDEX` supports three analyzers: LOG_ANALYZER (default, for machine-generated data), NO_OP_ANALYZER (exact match), PATTERN_ANALYZER (custom regex). Optimizes `SEARCH()`, `=`, `IN`, `LIKE`, `STARTS_WITH`. Index storage is typically 50–100% of indexed column size. Maintenance uses a free shared slot pool.

**Example**:
```sql
CREATE SEARCH INDEX log_idx
ON `project.dataset.app_logs`(message, source_ip)
OPTIONS (analyzer = 'LOG_ANALYZER');

SELECT * FROM app_logs
WHERE SEARCH(message, 'connection timeout 192.168.1.1');

-- Check index health
SELECT table_name, index_name, index_status, coverage_percentage
FROM `project.dataset.INFORMATION_SCHEMA.SEARCH_INDEXES`;
```

**Why It Matters**: On a 1 TB table, text search without an index scans the full terabyte. With an index, BigQuery reads only megabytes — dramatically reducing cost and latency.

**Reference**: https://cloud.google.com/bigquery/docs/search-index

### Wildcard tables have a 1,000-table limit and no caching

**Finding**: Wildcard queries (`_TABLE_SUFFIX`) match up to **1,000 tables**, cannot match views, and results are **never cached**. All matched tables must have compatible schemas. Always filter `_TABLE_SUFFIX` — without it, all matching tables are scanned.

**Example**:
```sql
-- Query date-sharded tables (common for GA4 exports)
SELECT event_type, user_id
FROM `my-project.analytics.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20260101' AND '20260131';

-- Migrate to partitioned table for better performance
CREATE TABLE events_partitioned
PARTITION BY DATE(event_timestamp) AS
SELECT *, PARSE_DATE('%Y%m%d', _TABLE_SUFFIX) AS shard_date
FROM `my-project.analytics.events_*`;
```

**Why It Matters**: Partitioned tables outperform sharded tables in query planning, support DML, and enable caching. Migration is recommended for any table queried frequently.

**Reference**: https://cloud.google.com/bigquery/docs/querying-wildcard-tables

---

## Conclusion

The highest-impact optimizations uncovered here fall into three categories. First, **avoiding silent full-table scans**: partition pruning fails with any function on the partition column, type mismatches, and subquery predicates — the scripting-variable workaround for dynamic partition filters is essential. Second, **materializing strategically**: non-recursive CTEs are never guaranteed materialized, so multi-referenced CTEs should become temp tables; HLL++ sketches should replace recurring COUNT(DISTINCT) queries; and GROUPING SETS should replace UNION ALL chains. Third, **leveraging 2024–2025 features**: pipe syntax (GA), History-Based Optimizations (automatic, zero-config), non-incremental materialized views, PK/FK constraint-driven join elimination, native JSON shredding, and search indexes all provide substantial gains with minimal migration effort. For Tableau specifically, the critical insight is that window functions must be pushed to BigQuery via Custom SQL — Tableau computes table calculations locally by default.