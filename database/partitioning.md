Partitioning splits one large table into smaller physical pieces — partitions — within the same database instance, while queries still address it as a single logical table. It's primarily a manageability and query-performance technique, distinct from sharding, which splits data across multiple separate database _instances_ for horizontal scale; a heavily-partitioned table can still outgrow a single machine, at which point sharding (not more partitioning) is the actual answer.

## 1. Why Partition a Table

Two motivations, and they often apply together:

- **Query performance via partition pruning**: if a query's `WHERE` clause can be matched to the partitioning key, the database only scans the relevant partition(s) instead of the entire table — turning a scan over 500 million rows into a scan over the 2 million rows in one month's partition.
- **Maintenance and lifecycle management**: dropping an entire old partition is an near-instant metadata operation; deleting the same rows via `DELETE FROM orders WHERE created_at < ...` is a slow, row-by-row operation that also bloats the table and its indexes until a vacuum/maintenance pass reclaims the space.

## 2. Partitioning Strategies

| Strategy | Splits by                                        | Good fit                                                                                                            |
| -------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| Range    | A range of values (dates, IDs)                   | Time-series data, append-mostly tables where old data is queried/archived by date range                             |
| List     | An explicit set of discrete values               | A `region` or `status` column with a small, known set of values                                                     |
| Hash     | A hash of the partitioning column, spread evenly | No natural range/list grouping exists, but you still want to split a huge table for maintenance/parallelism reasons |

Range partitioning is by far the most common in practice, specifically because most large operational tables are time-series-shaped (transactions, audit records, events) and "this month's data" is both the natural query pattern and the natural archival boundary.

## 3. Postgres Declarative Partitioning — A Working Example

```sql
CREATE TABLE transactions (
    id BIGINT NOT NULL,
    account_id BIGINT NOT NULL,
    amount NUMERIC(19,2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, created_at) -- the partition key must be part of any unique/primary key
) PARTITION BY RANGE (created_at);

CREATE TABLE transactions_2026_01 PARTITION OF transactions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE transactions_2026_02 PARTITION OF transactions
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

```sql
-- This query only touches the transactions_2026_02 partition — verify with EXPLAIN
EXPLAIN SELECT * FROM transactions WHERE created_at >= '2026-02-01' AND created_at < '2026-02-15';
-- Plan shows: "Seq Scan on transactions_2026_02" only — the January partition is never even opened
```

The application (and JPA/JDBI) still queries `transactions` as one table — partition pruning happens transparently inside the query planner, as long as the query's predicate actually references the partition key in a way the planner can use.

## 4. A Common Gotcha: the Partition Key Must Be in Every Unique Constraint

Postgres requires the partition key to be part of any primary key or unique constraint on a partitioned table — because uniqueness can only be enforced _within_ a partition, not globally across partitions without it.

```sql
-- This FAILS in Postgres: id alone can't be a primary key on a table partitioned by created_at
PRIMARY KEY (id)

-- This works: created_at (the partition key) is part of the primary key
PRIMARY KEY (id, created_at)
```

This has a real consequence for application code: if `id` was previously globally unique on its own (e.g., a JPA `@Id` auto-generated column), introducing range partitioning by a different column means the uniqueness guarantee now technically spans `(id, partitionKeyColumn)` — usually fine in practice since `id` generation (a sequence) still produces globally-unique values, but worth understanding rather than being surprised by when first partitioning an existing table.

## 5. Rolling Window Maintenance

The archival/retention pattern that makes range partitioning by date so useful in practice: create the next period's partition ahead of time, and drop (or detach and archive) the oldest one once it's outside the retention window.

```sql
-- Create next month's partition ahead of time (often automated via a scheduled job or an extension like pg_partman)
CREATE TABLE transactions_2026_03 PARTITION OF transactions
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

-- Drop a partition once outside the retention window — near-instant, unlike a row-by-row DELETE
DROP TABLE transactions_2026_01;

-- Or detach it first (keeps the data, just removes it from the partitioned table) for archival
ALTER TABLE transactions DETACH PARTITION transactions_2026_01;
```

For audit or regulatory data specifically, detach-and-archive to cold storage is usually preferred over an outright drop, since retention requirements often mean the data must still exist somewhere, just not in the actively-queried hot table.

## 6. When Partitioning Doesn't Help

- **Too many small partitions**: query planning overhead grows with partition count — thousands of tiny partitions can make planning slower than the pruning benefit is worth. Pick a partition granularity (monthly, not daily, for a moderate-volume table) sized to keep the partition count reasonable.
- **A partition key that doesn't match query patterns**: if queries rarely filter on the partitioning column, pruning never kicks in, and every query still scans every partition — worse than an unpartitioned table with a good index, since you've added complexity for no pruning benefit. Always verify pruning is actually happening with `EXPLAIN`, don't assume it from the schema alone.
- **Partitioning as a substitute for indexing**: partitioning narrows _which partition_ to scan; an index still determines how efficiently rows are found _within_ that partition. The two are complementary, not substitutes.

## 7. Best Practices

| Practice                                                                   | Recommendation                                                                                                                         |
| -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Choose the partition key to match your actual query patterns               | Pruning only helps if queries filter on the partition key — verify with `EXPLAIN`, don't assume.                                       |
| Default to range partitioning by date for time-series/append-mostly tables | The most common, best-understood case — aligns naturally with retention/archival needs.                                                |
| Automate rolling partition creation/retirement                             | Manually remembering to create next month's partition doesn't scale — use a scheduled job or an extension (e.g., `pg_partman`).        |
| Keep partition count reasonable                                            | Too many small partitions increase query planning overhead — size partitions by expected data volume, not an arbitrary fixed interval. |
| Remember the partition key must be part of any unique/primary key          | A partitioned table can't enforce global uniqueness independent of the partition key — plan schema/ID strategy accordingly.            |
| Detach and archive regulated/audit data rather than dropping it outright   | Retention requirements often mean the data must persist somewhere, just not in the hot table.                                          |
| Don't partition a table that doesn't need it                               | Partitioning adds real schema complexity — reach for it once table size genuinely causes query or maintenance pain, not preemptively.  |
