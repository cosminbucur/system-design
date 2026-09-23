# Database Composite Index Guide: Verification & Optimization

This guide covers how to check if a 3-column composite index is being utilized by your database engine and highlights critical mistakes developers make that prevent indexes from being used effectively.

---

## 1. How to Check Composite Index Utilization

To verify whether a query is hitting a composite index, inspect the query's **Execution Plan** (`EXPLAIN` or `EXPLAIN ANALYZE`).

### Example Setup

Assume an `orders` table with a composite index spanning three columns in exact order: `tenant_id`, `status`, and `created_at`.

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    tenant_id INT,
    status VARCHAR(20),
    created_at TIMESTAMP,
    total_amount DECIMAL(10, 2)
);

-- Composite Index on (col1, col2, col3)
CREATE INDEX idx_tenant_status_date 
ON orders (tenant_id, status, created_at);
```

### Ideal Query

```sql
SELECT * FROM orders 
WHERE tenant_id = 42 
  AND status = 'COMPLETED' 
  AND created_at >= '2026-01-01 00:00:00';
```

### Checking Execution Plans

#### PostgreSQL (`EXPLAIN ANALYZE`)
```sql
EXPLAIN ANALYZE 
SELECT * FROM orders 
WHERE tenant_id = 42 
  AND status = 'COMPLETED' 
  AND created_at >= '2026-01-01 00:00:00';
```
* **What to look for:**
  ```text
  Index Scan using idx_tenant_status_date on orders  (...)
    Index Cond: ((tenant_id = 42) AND ((status)::text = 'COMPLETED'::text) AND (created_at >= '2026-01-01 00:00:00'::timestamp))
  ```
* **Success Indicator:** The `Index Cond` block lists all three conditions (`tenant_id`, `status`, and `created_at`).

#### MySQL / MariaDB (`EXPLAIN`)
```sql
EXPLAIN SELECT * FROM orders 
WHERE tenant_id = 42 
  AND status = 'COMPLETED' 
  AND created_at >= '2026-01-01 00:00:00';
```
* **What to look for in tabular output:**

  | key | key_len | Extra |
  | :--- | :--- | :--- |
  | `idx_tenant_status_date` | `15` | `Using index condition` |

* **Success Indicator:** `key` displays the composite index name, and `key_len` equals the combined byte length of all 3 column data types (e.g., 4 bytes for INT + 7 bytes for VARCHAR + 4 bytes for TIMESTAMP = 15 bytes).

---

## 2. Common Developer Mistakes That Invalidate Composite Indexes

### Mistake 1: Wrapping an Indexed Column in a Function

Applying a function to an indexed column forces the database to evaluate every row, disabling the B-tree index lookup for that column.

* **Bad Query:**
  ```sql
  SELECT * FROM orders 
  WHERE tenant_id = 42 
    AND UPPER(status) = 'COMPLETED' 
    AND DATE(created_at) = '2026-09-23';
  ```
* **Why It Fails:** `UPPER(status)` and `DATE(created_at)` transform the column values at query runtime. The database can only use the index for `tenant_id`.
* **Fix:** Keep column names "clean" on the left side of comparison operators:
  ```sql
  SELECT * FROM orders 
  WHERE tenant_id = 42 
    AND status = 'COMPLETED' 
    AND created_at >= '2026-09-23 00:00:00' 
    AND created_at <  '2026-09-24 00:00:00';
  ```

---

### Mistake 2: Skipping the Leading Column (Leftmost Prefix Rule)

B-tree composite indexes require queries to filter starting from the first indexed column. Skipping leading columns prevents index traversal.

* **Bad Query:**
  ```sql
  SELECT * FROM orders 
  WHERE status = 'COMPLETED' 
    AND created_at >= '2026-09-23 00:00:00';
  ```
* **Why It Fails:** The index `(tenant_id, status, created_at)` is sorted primary-by-`tenant_id`. Without specifying `tenant_id`, the database cannot traverse the B-tree structure and falls back to a **Full Table Scan**.
* **Fix:** If filtering on `status` and `created_at` without `tenant_id` is common, create a secondary index:
  ```sql
  CREATE INDEX idx_status_date ON orders (status, created_at);
  ```

---

### Mistake 3: Placing a Range Condition Before Equality Columns

A composite B-tree index evaluates equality (`=`) conditions sequentially. However, **once a range condition (`>`, `<`, `BETWEEN`, `LIKE 'abc%'`) is hit, subsequent columns in the index cannot be evaluated via B-tree index scan**.

* **Bad Query:**
  ```sql
  SELECT * FROM orders 
  WHERE tenant_id = 42 
    AND created_at >= '2026-09-01 00:00:00' -- Range condition
    AND status = 'COMPLETED';               -- Equality condition placed after range
  ```
* **Why It Fails:** Index traversal stops at `created_at`. The database evaluates `tenant_id` and `created_at` using the index, but `status` must be evaluated post-lookup via row filtering.
* **Fix:** Structure your composite index order so that strict equality (`=`) columns come before range columns: `(tenant_id, status, created_at)`.

---

### Mistake 4: Implicit Type Mismatches

Passing a value whose data type does not match the column schema forces an implicit conversion, effectively wrapping the column in an internal function.

* **Bad Query:**
  ```sql
  -- Assuming tenant_id is an INTEGER column
  SELECT * FROM orders 
  WHERE tenant_id = '42' 
    AND status = 'COMPLETED';
  ```
* **Why It Fails:** In relational databases like PostgreSQL, comparing integer columns to string literals triggers an implicit cast (`CAST(tenant_id AS text)`), invalidating the index.
* **Fix:** Ensure query parameters match exact schema types:
  ```sql
  SELECT * FROM orders 
  WHERE tenant_id = 42 
    AND status = 'COMPLETED';
  ```

---

## 3. Quick Reference Summary

| Query Filter Pattern | Index Utilization Status |
| :--- | :--- |
| `WHERE tenant_id = 42 AND status = 'A' AND created_at >= Z` | **Full Index Scan** (Uses all 3 columns) |
| `WHERE tenant_id = 42 AND status = 'A'` | **Partial Index Scan** (Uses first 2 columns) |
| `WHERE tenant_id = 42 AND UPPER(status) = 'A'` | **Partial Index Scan** (Stops at `tenant_id`) |
| `WHERE status = 'A' AND created_at >= Z` | **Index Not Used** (Full table scan) |
| `WHERE tenant_id = 42 AND created_at >= Z AND status = 'A'` | **Partial Index Scan** (Stops at range filter) |