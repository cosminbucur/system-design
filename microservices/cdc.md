Change Data Capture (CDC) is a technique for reliably publishing every change made to a database as a stream of events, by reading the database's own transaction log rather than having application code explicitly publish an event alongside every write. It's the most common way to get events out of a transactional outbox table (or any table) in near real time, without polling the database and without touching the application code that performs the writes.

## 1. How It Works: Tailing the Transaction Log

Every relational database already maintains a durable, ordered log of every change made to it, for its own crash-recovery and replication purposes — Postgres calls this the write-ahead log (WAL), MySQL calls it the binlog. CDC tools attach to that log directly and stream each change out as an event.

```
Application writes a row → Database commits → Change appears in the WAL/binlog
                                                        ↓
                                        CDC connector (Debezium) reads the log
                                                        ↓
                                        Publishes a change event to Kafka
```

Because it reads the log the database was going to write anyway, CDC adds no extra write path, no additional table, and no polling query against the live database — it's a passive observer of changes that were already going to happen.

## 2. Debezium: The Standard Java-Ecosystem CDC Tool

Debezium is a set of Kafka Connect connectors, one per database engine, that implement this log-tailing approach and publish a structured change event per row change.

```json
{
  "before": null,
  "after": {
    "id": "ord-123",
    "customer_id": "cust-42",
    "status": "PLACED",
    "total": 149.99
  },
  "source": {
    "table": "orders",
    "db": "orders_db",
    "ts_ms": 1735689600000
  },
  "op": "c"
}
```

`op` identifies the kind of change: `c` (create), `u` (update), `d` (delete), `r` (an initial snapshot read). For an update, both `before` and `after` are populated, so a consumer can see exactly what changed, not just the new state — useful for anything that needs to react differently depending on which specific fields changed.

```java
@KafkaListener(topics = "orders_db.public.orders")
public void handleOrderChange(ChangeEvent event) {
    if ("c".equals(event.getOp())) {
        searchIndexService.index(event.getAfter()); // keep a search index in sync with the source table
    } else if ("d".equals(event.getOp())) {
        searchIndexService.remove(event.getBefore().getId());
    }
}
```

## 3. CDC vs. Polling: Why It's the Preferred Outbox Publisher

Both are ways to get events out of an outbox table (or any table) and onto a broker, but they differ meaningfully in cost and latency.

| | Polling publisher | CDC |
| --- | --- | --- |
| Mechanism | A scheduled job queries `WHERE processed = false` | Reads the database's transaction log directly |
| Latency | Bounded by the poll interval | Near real-time — events appear almost as soon as the transaction commits |
| Load on the database | Repeated polling queries against the live table | None — the log was already being written regardless |
| New infrastructure required | None beyond the existing database and a scheduler | A CDC connector (Debezium) plus Kafka Connect to run it |
| Ordering guarantees | Whatever the polling query's `ORDER BY` provides | Naturally ordered — the log is inherently ordered by commit sequence |

CDC is generally preferred at any real scale specifically because it removes both the polling load and the poll-interval latency, at the cost of running an additional piece of infrastructure (the connector) that a simpler polling job doesn't need.

## 4. CDC for More Than Just the Outbox Pattern

While CDC pairs naturally with the transactional outbox pattern, it's a general-purpose tool for anything that needs to react to database changes, not limited to a dedicated outbox table.

| Use case | What CDC provides |
| --- | --- |
| Outbox event publishing | Near real-time delivery of outbox rows without polling |
| Keeping a search index (Elasticsearch) in sync | Every row change in the source table flows into a reindex event automatically |
| Cache invalidation | A row change event triggers evicting the corresponding cache entry |
| Feeding a data warehouse / analytics pipeline | Every operational change streams into an analytical store without a nightly batch export |
| Legacy system integration | Capture changes from a database you can't modify the application code of, since CDC needs no application changes at all |

The last point is a distinctive strength: CDC works against *any* table, including ones owned by a legacy system or a third-party application where you have no ability to add "publish an event" code — the log-tailing approach needs no cooperation from the writing application at all.

## 5. What CDC Doesn't Give You for Free

CDC reliably captures *that* a row changed and *what* it changed to — it doesn't capture business intent. A change event says "the `status` column went from `PENDING` to `SHIPPED`," not "why," and it can't reconstruct a business-level event that never had a corresponding single-row change (e.g., "the customer's loyalty tier was recalculated based on their last 90 days of orders" isn't a change to any one row CDC can point to). For genuinely business-meaningful events with rich semantics, an application explicitly publishing a well-named domain event is often still clearer than inferring intent from a raw row diff — CDC shines specifically for "keep this other system in sync with this table's current state," less so for "notify the rest of the system that this specific business thing happened."

## 6. Initial Snapshot and Schema Changes

A CDC connector doesn't just capture changes going forward — it typically performs an initial snapshot of existing data first, so a downstream consumer building a search index or cache starts from a complete picture rather than only future changes.

```
Debezium startup:
  1. Snapshot: read the entire current table, emit as "r" (read) events
  2. Streaming: switch to tailing the log, emit ongoing "c"/"u"/"d" events from that point forward
```

Schema changes (adding a column, changing a type) on the source table need to be handled deliberately too — Debezium can capture DDL changes and propagate schema evolution to consumers, but a consumer's deserialization logic still needs to tolerate a table's shape changing over time, the same schema-evolution discipline that applies to any long-lived event contract.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Prefer CDC over polling for any real-scale outbox publishing | Removes both the polling load on the database and the poll-interval latency. |
| Use CDC for syncing derived stores (search index, cache), not for expressing business intent | A raw row-change event says what changed, not why — use explicit domain events for meaningful business occurrences. |
| Account for the initial snapshot when adding a new consumer | A consumer coming online needs the current full state, not just changes from that point forward. |
| Design consumers to tolerate schema evolution | The source table's shape will change over time — deserialization needs to handle added/changed fields gracefully. |
| Treat the CDC connector as real infrastructure needing its own monitoring | A stalled or lagging connector silently breaks every downstream consumer relying on it for freshness. |
| Don't couple tightly to the source table's exact internal schema | Consider a transformation step between raw CDC events and what consumers see, so internal schema changes don't ripple directly to every consumer. |
