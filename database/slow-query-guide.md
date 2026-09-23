# Database Query Optimization & Slow Query Log Review Guide

This document captures key strategies, workflows, and best practices for reviewing slow query logs, detecting unindexed lookups, eliminating full table scans, and optimizing expensive `JOIN` operations.

---

## Part 1: Reviewing Slow Query Logs

### 1. Enabling Verbose Slow Logging
To identify performance bottlenecks, ensure the database captures full execution metrics, parameter payloads, and unindexed scans.

* **MySQL / MariaDB:**
  ```sql
  SET GLOBAL slow_query_log = 'ON';
  SET GLOBAL long_query_time = 1; -- Log queries taking longer than 1s
  SET GLOBAL log_queries_not_using_indexes = 'ON'; -- Capture unindexed lookups
  ```
* **PostgreSQL:**
  ```ini
  # postgresql.conf
  log_min_duration_statement = 1000 # Log queries > 1000ms
  log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
  ```

---

### 2. Log Summarization Tools
Raw log files are often too noisy to process manually. Use dedicated tools to digest logs into actionable reports:

* **MySQL / MariaDB:** `pt-query-digest` (Percona Toolkit)
  ```bash
  pt-query-digest /var/log/mysql/mysql-slow.log > digest_report.txt
  ```
  *Key Metrics to Watch:* `No index used`, `Rows examined` vs `Rows sent`, and `Full scan`.

* **PostgreSQL:** `pgBadger` or `pg_stat_statements`
  ```bash
  pgbadger /var/log/postgresql/postgresql.log -o report.html
  ```
  Query `pg_stat_statements` directly to catch expensive queries and missing index patterns:
  ```sql
  SELECT query, calls, total_exec_time, rows, 
         100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS cache_hit_ratio
  FROM pg_stat_statements
  ORDER BY total_exec_time DESC
  LIMIT 10;
  ```

---

### 3. Payload Extraction & Execution Plan Analysis Workflow

1. **Extract Full Query Payload:** Capture exact parameters triggering slow log entries (e.g., specific filter values, large `IN(...)` lists, wide date ranges).
2. **Prepend `EXPLAIN ANALYZE`:** Execute the query using actual parameters (`EXPLAIN ANALYZE` in PostgreSQL/MySQL or `EXPLAIN FORMAT=JSON` in MySQL).
3. **Inspect Execution Tree Indicators:**
   * **Full Table Scan:** Look for `Seq Scan` (PostgreSQL) or `type: ALL` / `Using filesort` (MySQL) where `Rows Examined` matches total table rows.
   * **Unindexed Lookups:** Check for filter predicates applied *after* table fetch (`Filter: (col = 'payload')`).
   * **Expensive JOIN Operations:** Identify `Hash Join` disk spills or `Nested Loop` with extremely high iteration counts and execution time spikes.

---

## Part 2: Optimizing Expensive JOIN Operations

### 1. Indexing Join Keys & Filter Columns
* **Foreign Keys & Join Columns:** Maintain indexes on all columns appearing in `ON` clauses (`tbl_a.fk_id = tbl_b.id`).
* **Covering Indexes:** Add projected columns to indexes using `INCLUDE` to enable index-only scans and prevent table heap fetches:
  ```sql
  CREATE INDEX idx_orders_user_status ON orders (user_id, status) INCLUDE (total_amount);
  ```
* **Match Data Types Strictly:** Ensure joined columns match in type and collation (e.g., `INT` to `BIGINT` or `utf8mb4_bin` to `utf8mb4_unicode_ci` forces implicit conversions, disabling index lookups).

---

### 2. Filtering Early (Filter Pushdown)
Reduce the row count entering the join before performing the join step:

```sql
-- Less Efficient: Joins full tables, then filters
SELECT *
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at >= '2026-01-01';

-- Optimized: Limits order rows before joining
WITH recent_orders AS (
    SELECT id, user_id, total_amount
    FROM orders
    WHERE created_at >= '2026-01-01'
)
SELECT *
FROM recent_orders o
JOIN users u ON o.user_id = u.id;
```

---

### 3. Addressing Join Algorithm Bottlenecks

| Join Algorithm | Common Cause of Slowness | How to Fix |
| :--- | :--- | :--- |
| **Nested Loop Join** | Outer query loops for every row of the inner table when inner key lacks an index. | Add an index on the inner table's join key. Tune table statistics to allow switching to `Hash Join` for large datasets. |
| **Hash Join** | Large hash tables exceeding RAM limits and spilling to disk (`tempdb` / swap). | Increase memory allocation parameters (e.g., `SET work_mem = '64MB';` in PostgreSQL or tune `join_buffer_size` in MySQL). |
| **Sort-Merge Join** | Costly sorting phase required for non-indexed, unsorted datasets before merging. | Ensure join keys are indexed sequentially so inputs are pre-sorted. |

---

### 4. Architectural Interventions for High-Frequency Joins

* **Materialized Views:** Pre-calculate and store join results on disk, refreshing periodically or via triggers.
* **Summary / Rollup Tables:** Maintain aggregated table data for reporting instead of querying raw logs on demand.
* **Selective Denormalization:** Duplicate low-churn columns (e.g., storing `user_name` directly in `orders`) to eliminate joins in read-intensive hot paths.

---

## Part 3: Summary Matrix

| Issue | Log / EXPLAIN Indicator | Primary Remedy |
| :--- | :--- | :--- |
| **Unindexed Lookup** | `log_queries_not_using_indexes` flag / `Seq Scan` | Add covering index on `WHERE`, `JOIN`, or `ORDER BY` columns. |
| **Nested Loop Spills** | High execution time on `Nested Loop` with large inner row count | Add missing foreign key index; update stats to enable `Hash Join`. |
| **Full Table Scan** | `Rows Examined` >> `Rows Sent` | Run `ANALYZE table_name;` and remove implicit type casts breaking index usage. |
| **Memory Spill on Join** | Disk temporary file creation during `Hash Join` | Increase `work_mem` (PostgreSQL) or `join_buffer_size` (MySQL). |