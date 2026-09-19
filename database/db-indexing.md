A database index is a separate data structure that lets the database find rows matching a condition without scanning every row in the table — the same reason a book's index lets you find a topic without reading every page. Indexes are the single highest-leverage tool for query performance, but they're not free: every index speeds up the reads it matches while making every write to that table slightly slower, because the index itself has to be kept up to date too.

## 1. Why an Index Makes Queries Fast

Without an index, finding rows matching `WHERE customer_id = 42` means scanning every row in the table (`O(n)`) — a full table scan. An index on `customer_id` lets the database navigate directly to matching rows without looking at the rest of the table.

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- Before the index: sequential scan of the entire table
-- After the index: an index scan that jumps directly to matching rows
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

`EXPLAIN ANALYZE` (Postgres) or `EXPLAIN` (MySQL) shows exactly what the database actually did — always check this for a query that's slower than expected rather than assuming an index exists and is being used. A missing index, a wrong index, or a query written in a way the optimizer can't use the index it has (see §6) will all silently degrade to a full scan.

## 2. The B-Tree: The Default Index Structure

Most general-purpose indexes are a B-tree (balanced tree) — the structure keeps its data sorted and lets lookups, range scans, and ordered retrieval all run in `O(log n)` rather than `O(n)`.

```
Index on `created_at`:

              [2026-03-15]
             /            \
   [2026-01-10]          [2026-06-20]
   /         \            /         \
[rows...]  [rows...]   [rows...]  [rows...]
```

This is why a B-tree index is good not just for exact-match lookups (`WHERE id = 5`) but also for range queries (`WHERE created_at BETWEEN ... AND ...`) and for satisfying an `ORDER BY` without a separate sort step — the data is already stored in sorted order in the tree's leaf level.

## 3. Other Index Types

| Type | Structure | Best for |
| --- | --- | --- |
| B-tree | Balanced tree, sorted | Equality, range queries, ordering — the default, correct choice for most columns |
| Hash | Hash table of the indexed value | Pure equality lookups only (`=`), no range/ordering support — rarely worth choosing over B-tree in practice |
| GIN (Generalized Inverted Index) | Maps each element inside a composite value back to the rows containing it | Full-text search, JSONB containment queries, array `@>` queries |
| GiST (Generalized Search Tree) | Extensible tree supporting custom "overlaps"/"contains" operators | Geometric/geospatial data, range types |
| Bitmap index | A bitmap per distinct value | Very low-cardinality columns (e.g., a boolean or a small enum) in analytical/read-heavy workloads |

Reach for a non-B-tree index only when the query pattern genuinely doesn't fit one — a GIN index for a `JSONB` column being queried by key, for instance — rather than defaulting to it for ordinary equality/range lookups where a B-tree is simpler and just as fast.

## 4. Composite Indexes and Column Order

An index can span multiple columns, and the order of those columns is not a stylistic choice — it determines which query patterns the index can actually serve.

```sql
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
```

This index can efficiently serve:
- `WHERE customer_id = 42` (uses just the leading column)
- `WHERE customer_id = 42 AND status = 'SHIPPED'` (uses both columns)

But it **cannot** efficiently serve:
- `WHERE status = 'SHIPPED'` alone — the leading column (`customer_id`) isn't part of the filter, so the database can't narrow down where in the tree to look

The rule of thumb: put the column used for equality filtering first, and a column used for range filtering or sorting last — this is sometimes phrased as "equality, then range, then sort." Getting the order wrong doesn't cause an error; it just means the index silently isn't used for queries that don't lead with its first column, which is exactly the kind of problem `EXPLAIN` catches and a passing test suite won't.

## 5. Covering Indexes and Index-Only Scans

A covering index includes every column a query needs, so the database can answer the query entirely from the index itself without ever touching the underlying table row — an index-only scan.

```sql
-- Query only needs customer_id, order_date, and status
SELECT order_date, status FROM orders WHERE customer_id = 42;

-- Include order_date and status directly in the index so this query never touches the table
CREATE INDEX idx_orders_customer_covering ON orders(customer_id) INCLUDE (order_date, status);
```

This trades a larger index (it now stores more data) for avoiding a second read (the "table lookup" step that normally follows an index match) — worthwhile for hot, frequently-run queries where that avoided read is the difference between a fast and a merely-acceptable response time.

## 6. When an Index Silently Isn't Used

A query can fail to use an otherwise-correct index in ways that are easy to miss without checking `EXPLAIN`:

```sql
-- WRONG: wrapping the indexed column in a function prevents index use
-- unless a matching functional index also exists
SELECT * FROM orders WHERE LOWER(email) = 'a@b.com';

-- RIGHT: index the expression itself
CREATE INDEX idx_orders_email_lower ON orders(LOWER(email));
```

```sql
-- WRONG: a leading wildcard can't use a standard B-tree index at all
SELECT * FROM orders WHERE description LIKE '%order%';

-- A trailing wildcard CAN use a B-tree index
SELECT * FROM orders WHERE description LIKE 'order%';
```

Implicit type mismatches (comparing a `VARCHAR` column against an integer literal, forcing an implicit cast) are another common, easy-to-miss cause — the fix is always the same: check `EXPLAIN`, don't assume.

## 7. Indexes Aren't Free: The Write Cost

Every index on a table must be updated on every `INSERT`, `UPDATE` (of an indexed column), and `DELETE` — more indexes means more work per write, plus more storage.

| Cost | Why it happens |
| --- | --- |
| Slower writes | Every index needs its own entry inserted/updated/removed alongside the row itself |
| More storage | Each index is its own separate data structure, often comparable in size to the table itself for a wide index |
| More memory pressure | Indexes compete with table data for the database's buffer cache/shared memory |

This is why indexing "every column defensively" is a real mistake, not just wasted effort — index the columns actually used in `WHERE`, `JOIN`, and `ORDER BY` clauses for queries that matter, not every column a table happens to have.

## 8. Unique and Partial Indexes

A `UNIQUE` index both enforces a constraint and provides the performance benefit of a regular index for free — there's rarely a reason to add a plain index alongside a uniqueness constraint on the same column.

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

A partial index only indexes rows matching a condition, which is useful when queries only ever care about a subset of the table — smaller index, faster to maintain, and often a better fit than indexing the whole column.

```sql
-- Only active orders are ever looked up by this query pattern — no reason to index the rest
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'PENDING';
```

## 9. Cardinality: Why Indexing a Low-Cardinality Column Rarely Helps

An index's value comes from how much it narrows down the search — a column with very few distinct values (a boolean, a small enum with 3 states) doesn't narrow much, since a large fraction of the table still matches any given value. The query planner often correctly chooses a full table scan over such an index anyway, because reading the index and then jumping around the table for a large fraction of its rows can be slower than just scanning the table sequentially. High-cardinality columns (customer ID, email, an order number) are where indexes earn their cost most reliably.

## 10. Best Practices

| Practice | Recommendation |
| --- | --- |
| Verify with `EXPLAIN`/`EXPLAIN ANALYZE`, never assume | A missing index, wrong column order, or a query shape the optimizer can't use will silently fall back to a full scan. |
| Index columns actually used in `WHERE`/`JOIN`/`ORDER BY` | Don't index defensively — every extra index is a real, ongoing write-performance and storage cost. |
| Order composite index columns as equality, then range, then sort | The leading column determines which query patterns the index can serve at all. |
| Use covering indexes for hot, frequently-run queries | An index-only scan avoids the extra table lookup entirely — worth the larger index for queries that matter. |
| Prefer a `UNIQUE` index over a constraint plus a separate plain index | It enforces uniqueness and provides the read performance benefit in one structure. |
| Use partial indexes when queries only ever target a subset of rows | Smaller, cheaper to maintain, and often a better fit than indexing the entire column. |
| Avoid indexing low-cardinality columns in isolation | Few distinct values rarely narrow the search enough to beat a sequential scan — consider a composite index instead. |
| Watch for functions/casts wrapping an indexed column in a query | These silently prevent standard index use unless a matching functional index exists. |
