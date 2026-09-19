SQL (relational) and NoSQL databases solve the same underlying problem — durable, queryable storage — with different tradeoffs around schema rigidity, consistency, and scaling. Neither is universally "better"; the right choice depends on your data's shape and your consistency requirements.

## 1. The Relational Model — Core Concepts

Data is organized into tables (rows/columns) with a fixed schema, and relationships between tables are expressed via foreign keys rather than nesting data inside a single record.

```sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    total NUMERIC(10,2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

The database itself enforces referential integrity (`REFERENCES customers(id)` — you can't insert an order for a customer that doesn't exist) and constraints (`UNIQUE`, `NOT NULL`) — correctness rules live in the schema, not just in application code.

## 2. Normalization — And When to Break It

Normalization organizes data to eliminate redundancy: each fact is stored in exactly one place, referenced elsewhere by key. The classic normal forms build on each other:

| Form | Rule                                                 | Example violation it fixes                                                                                             |
| ---- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 1NF  | Each column holds atomic values, no repeating groups | A `phone_numbers` column storing `"555-1234, 555-5678"` as one string                                                  |
| 2NF  | No partial dependency on part of a composite key     | An `order_items` table storing `product_name` (depends only on `product_id`, not the full `order_id + product_id` key) |
| 3NF  | No transitive dependency on non-key columns          | Storing `customer_city` on the `orders` table when it's really an attribute of `customer_id`, not of the order itself  |

Normalization prevents update anomalies (change a customer's email in one place, forget to update three denormalized copies elsewhere) — but every join required to reassemble normalized data costs a query. **Denormalization** — deliberately duplicating data to avoid joins — is a valid, deliberate performance tradeoff for read-heavy paths (this is exactly what a CQRS read model does), not a mistake as long as you've explicitly decided which copy is the source of truth and how the duplicates stay in sync.

## 3. ACID — What a Relational Transaction Guarantees

| Property    | Guarantee                                                                                |
| ----------- | ---------------------------------------------------------------------------------------- |
| Atomicity   | A transaction's statements all succeed or all roll back — no partial writes              |
| Consistency | A transaction moves the database from one valid state to another, respecting constraints |
| Isolation   | Concurrent transactions don't see each other's uncommitted intermediate state            |
| Durability  | Once committed, data survives a crash                                                    |

```java
@Transactional
public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();
    from.withdraw(amount);
    to.deposit(amount);
    // both updates commit together, or neither does — Atomicity in action
}
```

## 4. Isolation Levels — The Real Tradeoff Inside ACID

"Isolation" isn't all-or-nothing — SQL databases let you choose how much concurrent transactions can see of each other, trading correctness guarantees against throughput.

| Level                           | Prevents                     | Still allows                                                                                   |
| ------------------------------- | ---------------------------- | ---------------------------------------------------------------------------------------------- |
| Read Uncommitted                | Nothing                      | Dirty reads (seeing another transaction's uncommitted changes)                                 |
| Read Committed (common default) | Dirty reads                  | Non-repeatable reads (same query, run twice in one transaction, returns different results)     |
| Repeatable Read                 | Dirty + non-repeatable reads | Phantom reads (a new row matching your `WHERE` appears on a re-query)                          |
| Serializable                    | All of the above             | Nothing — transactions behave as if run one at a time; highest correctness, lowest concurrency |

Higher isolation reduces concurrency (more locking, more chance of contention/deadlock) — most applications run at `Read Committed` and reach for optimistic locking (a `@Version` field) for the specific operations that actually need stronger guarantees, rather than raising the isolation level globally.

## 5. Indexing — Why Queries Are Fast (Or Aren't)

Without an index, the database scans every row to find matches (`O(n)`); an index (typically a B-tree) lets it jump directly to matching rows (`O(log n)`).

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_created_at_id ON orders(created_at DESC, id DESC); -- for cursor pagination
```

Indexes aren't free: they speed up reads but slow down writes (every insert/update must also update each index) and consume storage — index the columns you actually filter/sort/join on, not every column defensively. `EXPLAIN ANALYZE` (Postgres) or `EXPLAIN` (MySQL) shows whether a query is actually using an index or falling back to a full table scan — always check this for a query that's slower than expected, rather than guessing.

## 6. NoSQL — Why It Exists

NoSQL databases relax the relational model's rigidity (fixed schema, joins, strict ACID) in exchange for horizontal scalability and flexibility for data shapes that don't map cleanly onto tables. There isn't one "NoSQL model" — it's an umbrella over several distinct approaches:

| Type                        | Model                                                                            | Example                         | Good fit for                                                                                    |
| --------------------------- | -------------------------------------------------------------------------------- | ------------------------------- | ----------------------------------------------------------------------------------------------- |
| Document                    | JSON-like documents, flexible schema per document                                | MongoDB, Couchbase              | Content with varying/nested structure (product catalogs with different attributes per category) |
| Key-value                   | Simple key → opaque value lookup                                                 | Redis, DynamoDB (partition key) | Caching, session storage, very high-throughput simple lookups                                   |
| Column-family (wide-column) | Rows can have different columns; optimized for huge datasets with sparse columns | Cassandra, HBase                | Time-series data, huge write volumes, data naturally partitioned by a key                       |
| Graph                       | Nodes and edges, optimized for traversing relationships                          | Neo4j                           | Highly interconnected data — social graphs, fraud detection, recommendation engines             |

## 7. Document Databases in Java (MongoDB Example)

```java
public record Product(String sku, String name, BigDecimal price, Map<String, Object> attributes) {}

// Spring Data MongoDB — similar repository pattern to Spring Data JPA
public interface ProductRepository extends MongoRepository<Product, String> {
    List<Product> findByAttributesContaining(String key, Object value);
}
```

```json
// Two products in the same collection can have entirely different attribute shapes — no migration required
{ "sku": "SKU-1", "name": "T-Shirt", "attributes": { "size": "M", "color": "blue" } }
{ "sku": "SKU-2", "name": "Laptop", "attributes": { "ramGb": 16, "cpu": "M3" } }
```

This flexibility is also the tradeoff: the database no longer enforces that every product has consistent fields — that validation responsibility moves into application code (or a schema-validation layer MongoDB also offers), rather than being guaranteed by the storage engine itself.

## 8. Key-Value and Wide-Column at Scale (DynamoDB Example)

DynamoDB (and similarly-modeled stores) are designed around **access patterns first** — you design your key structure around the specific queries you'll run, rather than normalizing data and letting arbitrary joins happen at query time (which these databases don't support well or at all).

```java
// AWS SDK v2 — DynamoDB Enhanced Client
@DynamoDbBean
public class Order {
    private String customerId; // partition key — determines which physical partition stores this item
    private String orderId;    // sort key — orders for one customer are stored together, sorted by orderId

    @DynamoDbPartitionKey
    public String getCustomerId() { return customerId; }

    @DynamoDbSortKey
    public String getOrderId() { return orderId; }
}
```

Getting the partition key wrong is the classic DynamoDB/Cassandra mistake: a poorly chosen key creates a "hot partition" (all traffic hitting one physical node) instead of spreading load evenly — this is the NoSQL analogue of choosing a Kafka partition key, and the same principle applies: pick a key that spreads load but keeps related data (that you'll query together) co-located.

## 9. CAP Theorem — The Real Constraint Behind "NoSQL Scales Better"

In the presence of a network partition (P, which is unavoidable in any distributed system), you must choose between Consistency (every read sees the latest write) and Availability (every request gets a response, even if it might be stale) — you can't have both during a partition.

| Choice                                  | Behavior during a partition                                   | Example systems                                                                       |
| --------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| CP (Consistency + Partition tolerance)  | Refuses/delays requests rather than risk returning stale data | Traditional RDBMS in a synchronous replication setup, MongoDB (default config), HBase |
| AP (Availability + Partition tolerance) | Keeps responding, but different nodes might briefly disagree  | Cassandra, DynamoDB (default config)                                                  |

Most systems aren't purely one or the other — they let you tune the tradeoff per operation (e.g., DynamoDB's "strongly consistent" vs. "eventually consistent" reads). The practical question to ask isn't "is this database CP or AP" as a trivia fact — it's "for this specific piece of data, can my business logic tolerate briefly reading stale data, or does it need a guaranteed-current read?" (the same staleness-tolerance question already raised for caching).

## 10. Choosing SQL vs NoSQL

| Signal                                                                                               | Leans toward                |
| ---------------------------------------------------------------------------------------------------- | --------------------------- |
| Data has clear relationships, needs multi-table joins/queries                                        | SQL                         |
| Strong consistency and ACID transactions across multiple entities are required                       | SQL                         |
| Schema is well-understood and stable                                                                 | SQL                         |
| Data is naturally document-shaped, schema varies per record, or nests deeply                         | Document NoSQL              |
| Extremely high write throughput, horizontal scale beyond what a single relational primary can handle | Wide-column/key-value NoSQL |
| Access patterns are known upfront and few in number (classic NoSQL data modeling constraint)         | Key-value/wide-column NoSQL |
| Data is fundamentally about relationships/traversal (who's connected to whom, shortest path)         | Graph                       |

Many real systems are **polyglot persistence**: a relational database for the core transactional data, Redis for caching and sessions, and perhaps a document store for a specific service whose data genuinely doesn't fit tables well — this maps naturally onto "database per service", where each bounded context picks the storage technology that fits its own data shape, rather than forcing one database technology on the whole system.

## 11. Best Practices

| Practice                                                                                     | Recommendation                                                                                                              |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Default to a relational database unless you have a specific reason not to                    | ACID guarantees and mature tooling solve most application needs without NoSQL's tradeoffs.                                  |
| Denormalize deliberately, not accidentally                                                   | Duplicate data for read performance only when you've decided which copy is the source of truth and how copies stay in sync. |
| Check `EXPLAIN`/`EXPLAIN ANALYZE` before assuming an index is being used                     | A missing or wrong index silently degrades a query to a full table scan.                                                    |
| Choose isolation level per real requirement, not by raising it globally "to be safe"         | Higher isolation trades away concurrency — reach for optimistic locking on the specific operations that need it instead.    |
| Design NoSQL key structure around actual access patterns, not habit from relational modeling | A DynamoDB/Cassandra table designed like a normalized SQL table usually performs badly and creates hot partitions.          |
| Ask "can this specific data tolerate staleness," not "is this database CP or AP"             | The CAP tradeoff is made per read/write, not as a one-time database-wide fact.                                              |
| Embrace polyglot persistence per bounded context, not one database for everything            | Let each service pick the storage model that fits its own data shape.                                                       |
